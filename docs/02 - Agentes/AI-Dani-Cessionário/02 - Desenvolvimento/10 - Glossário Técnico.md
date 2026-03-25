# 10 - Glossário Técnico

| **Destinatário** | **Escopo** | **Versão** | **Responsável** | **Data da versão** |
|---|---|---|---|---|
| Todo o time (produto, engenharia, design) | Glossário de termos do domínio e técnicos do AI-Dani-Cessionário | v1.0 | Claude Code Desktop | 23/03/2026 (America/Fortaleza) |

---

> 📌 **TL;DR**
>
> - Fonte única de verdade para terminologia do agente AI-Dani-Cessionário.
> - Termos de domínio (negócio imobiliário + plataforma) + termos técnicos do agente.
> - A partir do D12, todo documento subsequente usa os termos exatamente como definidos aqui.
> - Termos em inglês aparecem apenas quando são consagrados na área técnica.
> - Ambiguidades frequentes resolvidas na seção de anti-exemplos.

---

## 1. Termos de Domínio (Negócio)

| Termo | Definição | Fonte |
|---|---|---|
| **Cedente** | Proprietário original do contrato imobiliário que deseja repassar. Dados pessoais nunca são expostos ao Cessionário na plataforma. | D01 |
| **Cessionário** | Investidor ou comprador que adquire o repasse imobiliário. Usuário-alvo da Dani. | D01 |
| **Δ (Delta)** | Diferença monetária entre a Tabela Atual e a Tabela Contrato de uma oportunidade. Fórmula: `Δ = Tabela Atual − Tabela Contrato`. Base de cálculo da Comissão Comprador quando Δ > 0. | D01 |
| **Tabela Atual** | Preço vigente do imóvel conforme a tabela de preços da incorporadora no momento da análise. Pode ser superior ou inferior à Tabela Contrato. | D01 |
| **Tabela Contrato** | Preço do imóvel na data do contrato original assinado pelo Cedente com a incorporadora. Valor histórico fixo. | D01 |
| **Valor Pago pelo Cedente** | Somatório de parcelas já pagas pelo Cedente à incorporadora até o momento do repasse. Usado como base da Comissão quando Δ ≤ 0. | D01 |
| **Comissão Comprador** | Taxa cobrada do Cessionário pela plataforma. Fórmula padrão: `20% × Δ` (quando Δ > 0). Fallback: `20% × Valor Pago pelo Cedente` (quando Δ ≤ 0). Descontos por faixa são aplicados exclusivamente pelo Admin. | D01 |
| **Escrow** | Conta garantia onde o Cessionário deposita o Custo Total (Preço Repasse + Comissão) após o aceite da proposta. Prazo padrão: 10 dias úteis. Extensão possível de +5 dias úteis mediante aprovação. | D01 |
| **Custo Total** | Valor total que o Cessionário deposita no Escrow. Fórmula: `Preço Repasse + Comissão Comprador`. | D01 |
| **ROI (Return on Investment)** | Retorno projetado sobre o capital investido. Fórmula: `(Tabela Atual − Custo Total) ÷ Custo Total × 100`. Apresentado em 3 cenários: conservador, base e otimista. | D01 |
| **OPR-XXXX-XXXX** | Código único de identificação de uma oportunidade no marketplace. Formato padronizado. Exemplo: `OPR-2024-0042`. | D01 |
| **Oportunidade** | Oferta de repasse publicada no marketplace. Possui estados: Disponível, Em negociação, Encerrada. | D01 |
| **Repasse** | Transferência do contrato imobiliário do Cedente para o Cessionário, incluindo as obrigações e direitos do contrato original com a incorporadora. | D01 |
| **Dossiê** | Conjunto de documentos obrigatórios para validação do repasse: contrato original, matrícula do imóvel, certidões. | D01 |
| **KYC (Know Your Customer)** | Processo de verificação de identidade obrigatório para o Cessionário antes de enviar propostas. Documentos: documento de identidade (frente e verso), selfie com verificação de vivacidade, comprovante de endereço ≤ 90 dias. Prazo: ≤ 30min (automático) ou ≤ 2 dias úteis (manual). | D01 |
| **Envelope ZapSign** | Pacote de assinatura eletrônica enviado ao Cessionário para formalização do contrato de cessão. Prazo de 5 dias úteis para assinar. | D01 |
| **Score de Risco** | Avaliação de risco de uma oportunidade em escala de 1 a 10, calculada pelo agente com base em dados do marketplace. 1–3: risco baixo. 4–6: risco moderado. 7–10: risco alto. | D01 |
| **Portfólio** | Conjunto de oportunidades que um Cessionário está analisando ou propõe adquirir simultaneamente. Usado na análise de distribuição de capital. | D01 |

---

## 2. Termos do Agente

| Termo | Definição | Fonte |
|---|---|---|
| **AI-Dani-Cessionário** | Nome interno do produto. O agente de IA especializado em análise de oportunidades para o Cessionário na plataforma Repasse Seguro. | D01 |
| **Dani** | Nome exibido na interface para o Cessionário. Persona: Analista de Oportunidades. | D01 |
| **Calculadora de Comissão** | Módulo determinístico do sistema que calcula comissão, Escrow e ROI usando fórmulas fixas. Não depende do modelo de IA. Funciona como fallback primário quando a Dani está indisponível. | D01, D05 |
| **Fallback** | Estado de operação da Calculadora quando a Dani (IA) está indisponível. Apresenta cálculos determinísticos sem análise contextual. | D01, D05 |
| **Takeover** | Intervenção manual do Admin em uma conversa quando a confiança da Dani está abaixo do threshold configurado (padrão: 80%). O Admin assume a conversa diretamente. | D01 |
| **Isolamento de dados** | Princípio de segurança que garante que a Dani opera exclusivamente com dados do Cessionário autenticado. Implementado em 3 camadas: filtro de escopo, filtro de contexto, reforço no system prompt. | D01, D05 |
| **Conversation starters** | Sugestões de conversa exibidas no chat quando aberto sem contexto de oportunidade. 4 chips de ação rápida que enviam a mensagem automaticamente ao serem clicados. | D01, D06 |
| **FAB (Floating Action Button)** | Botão flutuante fixo em todas as telas do módulo Cessionário que abre o chat da Dani. Exibe badge numérica quando há alertas não lidos. | D06 |
| **Alerta proativo** | Notificação gerada pela Dani para o Cessionário sobre mudanças relevantes no marketplace (nova oportunidade, oportunidade encerrada, atualização de negociação). | D05 |
| **System prompt** | Instrução permanente fornecida ao modelo de IA que define a persona, as restrições de dados e o comportamento da Dani. Armazenado como código, versionado. | D02, D05 |
| **Streaming** | Técnica de entrega de respostas da Dani em tempo real, token a token, via SSE (Server-Sent Events). Melhora percepção de latência. | D02 |
| **OTP (One-Time Password)** | Código de uso único com 6 dígitos, com prazo de 15 minutos, usado para vincular o número de WhatsApp ao perfil do Cessionário. Rate limit: 3 tentativas/hora. Hard block: 5 falhas consecutivas. | D01 |
| **RBAC (Role-Based Access Control)** | Controle de acesso por perfil. Cada Cessionário acessa somente os próprios dados. Implementado via `CessionarioOwnerGuard` no NestJS. | D01, D02 |

---

## 3. Termos Técnicos (Stack e Infraestrutura)

| Termo | Definição | Fonte |
|---|---|---|
| **NestJS** | Framework HTTP Node.js utilizado no backend da Dani. Arquitetura modular com decorators TypeScript, guards e pipes. | D02 |
| **Prisma** | ORM usado para acesso ao banco de dados PostgreSQL. Único ORM aprovado pela ShiftLabs. | D02 |
| **Supabase** | Plataforma de banco de dados gerenciado (PostgreSQL + Auth + Storage + Realtime + pgvector). Banco de dados da Dani. | D02 |
| **Redis** | Banco de dados em memória usado para cache de rate limiting e estado operacional do agente. TTL obrigatório em toda chave. | D02 |
| **RabbitMQ** | Sistema de filas para processamento assíncrono de notificações e mensagens WhatsApp. Toda fila com DLQ. | D02 |
| **GPT-4 (`gpt-4o`)** | Modelo de linguagem grande (LLM) usado como cérebro da Dani. Fornecido pela OpenAI. | D02 |
| **Langfuse** | Ferramenta de observabilidade de LLM. Registra traces, latência, custo e qualidade das respostas da Dani. | D02 |
| **pgvector** | Extensão PostgreSQL para armazenamento e busca vetorial. Usada para RAG (histórico de conversas e dados de empreendimentos). | D02 |
| **SSE (Server-Sent Events)** | Protocolo de comunicação unidirecional do servidor para o cliente. Usado para streaming das respostas da Dani. | D02 |
| **JWT (JSON Web Token)** | Token de autenticação stateless. A Dani herda o JWT da sessão da plataforma Repasse Seguro sem exigir novo login. | D02 |
| **EvolutionAPI** | Gateway WhatsApp Business open-source. Usado na Fase 2 para o canal WhatsApp da Dani. | D02 |
| **Framer Motion** | Biblioteca de animações para React. Usada para todas as animações da interface da Dani (FAB, chat window, bubbles). | D02, D04 |
| **shadcn/ui** | Biblioteca de componentes de UI com ownership total do código. Base dos componentes visuais da Dani. | D02, D03 |
| **Tailwind CSS v4** | Framework CSS utility-first. Estilização de toda a interface da Dani. | D02, D03 |
| **TanStack Query** | Biblioteca de gerenciamento de estado do servidor. Usado para sincronização de dados do chat no frontend. | D02 |
| **Zustand** | Biblioteca de estado global de UI. Usado para estado do chat (aberto/fechado, status da Dani). | D02 |
| **DLQ (Dead-Letter Queue)** | Fila de mensagens que falharam no processamento após retries esgotados. Toda fila RabbitMQ tem uma DLQ associada. | D02 |
| **RAG (Retrieval-Augmented Generation)** | Técnica que combina busca vetorial com geração de texto. Usado pela Dani para contextualizar respostas com histórico de conversas. | D02 |
| **ADR (Architecture Decision Record)** | Documento que registra decisões técnicas que desviam do padrão ShiftLabs. Obrigatório para todo desvio de stack. | D02 |

---

## 4. Anti-exemplos (Termos que Geram Confusão)

| Confusão | Correto | Errado |
|---|---|---|
| "IA" vs "Dani" | "Dani" é a persona exibida ao usuário. "IA" ou "AI" é o modelo subjacente. Nunca expor "IA" ao usuário final — sempre "Dani". | "A IA calculou..." (no chat ao usuário) |
| "Comissão" vs "Comissão Comprador" | Internamente: "Comissão Comprador" para a taxa paga pelo Cessionário. "Comissão" pode ser ambíguo. | "Comissão da plataforma" (vago) |
| "Taxa" vs "Comissão" | O documento usa "Comissão Comprador". "Taxa" não é o termo do domínio. | "Taxa de serviço de 20%" |
| "Investimento" vs "Custo Total" | "Custo Total" = Preço Repasse + Comissão = valor do Escrow. "Investimento" é informal e impreciso no contexto técnico. | "Valor do investimento" (no código) |
| "Calculadora" vs "Dani" | A Calculadora é o módulo determinístico (sem IA). A Dani é o agente de IA. São módulos separados. | "A Dani calculou a comissão" (quando foi a Calculadora no fallback) |
| "Sessão" vs "Conversa" | "Conversa" = registro persistente (`dani_conversas`). "Sessão" = estado temporário de autenticação (`dani_sessoes`). | Usar "sessão" e "conversa" como sinônimos |

---

## Changelog

| Data | Versão | Descrição |
|---|---|---|
| 23/03/2026 | v1.0 | Versão inicial. 18 termos de domínio, 13 termos do agente, 16 termos técnicos, 6 anti-exemplos. Fonte única de verdade para terminologia do AI-Dani-Cessionário. |
