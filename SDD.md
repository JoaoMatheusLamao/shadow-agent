# SDD.md — Software Design Document
**Projeto:** Middleware/Oráculo de IA para Suporte ao Cliente
**Status:** Stack e decisões-chave de MVP definidas — plataforma de suporte pendente
**Metodologia:** Spec-Driven Development (GitHub Spec Kit)

> Este documento segue a estrutura recomendada pelo Spec Kit (`specify`), adaptada
> para os artefatos deste projeto: Constitution → Spec → Plan → Tasks. As seções
> abaixo mapeiam para essas fases. Ao rodar os comandos do Spec Kit
> (`/constitution`, `/specify`, `/plan`, `/tasks`) neste repositório, este arquivo
> deve ser tratado como a fonte de verdade arquitetural que alimenta o `/plan`.

---

## 0. Constitution (princípios não negociáveis)

Ver `CLAUDE.md`, seção 3, para a lista completa de princípios de engenharia
(resiliência de webhook, processamento assíncrono sem fila, LGPD, Clean
Architecture). Este SDD assume esses princípios como restrições de design, não
como sugestões.

---

## 1. Objetivo do Sistema

Construir um serviço backend MVP que se posiciona entre uma plataforma de
atendimento de mercado e uma LLM com RAG, de forma que:
- A plataforma de suporte não perceba diferença entre uma resposta gerada por IA e
  uma resposta manual (em termos de formato/latência aceitável).
- O sistema seja capaz de trocar de plataforma de suporte ou de modelo de LLM
  (via OpenRouter) sem reescrever regra de negócio.
- Toda a operação seja auditável e conforme à LGPD, com retenção de dados
  definida (90 dias).
- A infraestrutura de sustentação seja a mais simples e barata possível para
  este estágio (sem fila dedicada, sem múltiplos serviços gerenciados).

---

## 2. Decisão de Stack: Node.js/TypeScript vs Python

### 2.1 Critérios de avaliação

| Critério | Peso | Justificativa |
|---|---|---|
| Concorrência em I/O (webhooks simultâneos) | Médio | MVP tem volume baixo/moderado — menos crítico que em escala |
| Latência de resposta ao webhook (ACK) | Alto | Plataformas de suporte costumam ter timeout curto (2–10s) para resposta do endpoint |
| Ecossistema de IA/RAG | Alto | Velocidade de construção do pipeline de RAG é crítica para validar o MVP rápido |
| Simplicidade operacional (sem fila) | Alto | Decisão de MVP: processamento in-process, sem broker externo |
| Tipagem e manutenibilidade em Clean Architecture | Médio | Projeto de longa duração, múltiplos adapters futuros |
| Velocidade de desenvolvimento inicial | Alto | Time-to-first-integration importa para validar o produto |
| Custo operacional (hosting, nº de serviços) | Alto | Restrição explícita do MVP: barato de manter |

### 2.2 Node.js / TypeScript

**Concorrência em I/O**
Node.js usa um event loop single-threaded não bloqueante, correspondência
natural para muitas conexões webhook concorrentes majoritariamente esperando
I/O. Em um MVP de volume baixo/moderado, essa vantagem existe mas é menos
decisiva do que em um cenário de alta escala.

**Tempo de resposta de webhooks**
Fastify atinge overhead de roteamento muito baixo, favorecendo o padrão "ACK
imediato, processa depois" mesmo sem fila — via `setImmediate`/task assíncrona
in-process.

**Ecossistema de IA/RAG**
Competitivo (SDKs oficiais, Vercel AI SDK, LangChain.js), mas historicamente
menos maduro que Python para prototipagem rápida de pipelines de RAG
(chunking, retrieval híbrido, avaliação de qualidade).

**Riscos**
- Ecossistema de RAG mais "montado à mão" comparado a Python — maior tempo até
  o primeiro pipeline funcional, o que pesa contra a prioridade de velocidade
  do MVP.

### 2.3 Python

**Concorrência em I/O**
FastAPI + `async`/`await` (ASGI, via Uvicorn) alcança um modelo de concorrência
comparável ao Node para I/O-bound, desde que o código do caminho crítico use
bibliotecas assíncronas (`httpx.AsyncClient`, `asyncpg`). Sem fila dedicada e
com volume de MVP, o risco de bloquear o event loop é presente mas gerenciável
com disciplina simples — não exige a robustez de um sistema de alta
concorrência.

**Tempo de resposta de webhooks**
FastAPI + `BackgroundTasks` cobre diretamente o padrão "ACK imediato, processa
depois" sem exigir infraestrutura de fila — exatamente o modelo escolhido para
este MVP (ver `CLAUDE.md` §3.2).

**Ecossistema de IA/RAG**
Ponto forte decisivo para este projeto: LangChain, LlamaIndex, clientes de
embeddings e utilitários de chunking prontos aceleram a validação do pipeline
de RAG — a prioridade nº1 declarada para o MVP.

**Simplicidade operacional**
FastAPI + Postgres/pgvector cobre HTTP, persistência e busca vetorial com um
único banco gerenciado. Nenhum broker de fila é necessário.

**Riscos**
- Requer disciplina para manter o caminho crítico assíncrono (evitar chamadas
  síncronas bloqueantes dentro de handlers `async`) — mitigado por volume de
  MVP ser baixo/moderado e por lint/type-check (`ruff`, `mypy`) cobrirem parte
  do risco.

### 2.4 Decisão Final — MVP (FINALIZADA)

**Contexto que resolve a decisão:** o projeto é um MVP com prioridade explícita
em velocidade de entrega e baixo custo operacional. Filas dedicadas
(BullMQ/Celery) foram descartadas do escopo inicial — processamento acontece
in-process, em background, no mesmo serviço que recebe o webhook.

Sob essa restrição, o fator decisivo deixa de ser "disciplina de assincronia em
produção sob alta concorrência" (onde Node tem vantagem natural) e passa a ser
**velocidade de construção do pipeline de RAG**, o ponto forte mais claro do
ecossistema Python. Como o volume de mensagens de um MVP é, por definição,
baixo/moderado, o risco de bloquear o event loop é aceitável neste estágio.

**Decisão final:** Python 3.12 + FastAPI + Postgres/pgvector + BackgroundTasks
(sem fila dedicada) + OpenRouter como provedor de LLM.
**Data:** 2026-07-19
**Responsável:** usuário do projeto

**Gatilho de reavaliação:** se o volume de mensagens ou os requisitos de
confiabilidade de entrega crescerem a ponto de a perda ocasional de mensagem em
crash do processo deixar de ser aceitável, reavaliar introdução de
Celery+Redis (mesma linguagem, sem necessidade de reescrever a lógica de
negócio, dado o isolamento via `QueuePort` já previsto na Clean Architecture —
ver seção 3.3).

---

## 3. Arquitetura de Alto Nível

### 3.1 Fluxo de Dados (MVP — sem fila dedicada)

```
┌──────────────────┐
│  Plataforma de    │   (a definir — Zendesk / Blip / Intercom)
│  Suporte          │
└─────────┬─────────┘
          │ (1) POST /webhooks/{platform}
          ▼
┌──────────────────────────────────────────────┐
│  FastAPI — Camada de Entrada (Adapter)         │
│  - Valida assinatura/HMAC do webhook           │
│  - INSERT em processed_events (dedup por       │
│    externalEventId, UNIQUE constraint)         │
│    → se já existe: retorna 200 e encerra       │
│  - Converte payload para modelo de domínio     │
│  - Dispara BackgroundTasks.add_task(processar) │
│  - Retorna 202 Accepted IMEDIATAMENTE          │
└─────────┬──────────────────────────────────────┘
          │ (2) task em background, mesmo processo
          ▼
┌──────────────────────────────────────────────┐
│  Camada de Aplicação — ProcessIncomingMessage  │
│                                                │
│  (2a) RetrieveContext                          │
│    ┌────────────────────────────────────┐     │
│    │  pgvector (mesmo Postgres)          │     │
│    │  + APIs internas do cliente         │     │
│    │    (CRM, pedidos, faturas)          │     │
│    │  retry manual c/ backoff curto      │     │
│    └────────────────────────────────────┘     │
│                                                │
│  (2b) GenerateResponse                         │
│    ┌────────────────────────────────────┐     │
│    │  LLM via OpenRouter                 │     │
│    │  + Persona / prompt de sistema      │     │
│    │  retry manual c/ backoff curto      │     │
│    └────────────────────────────────────┘     │
│                                                │
│  (2c) Validação/Sanitização da resposta        │
│    - Se falha após retries → grava em          │
│      failed_messages (auditoria/replay manual) │
│    - Se confiança baixa → marca p/ escalonar   │
└─────────┬──────────────────────────────────────┘
          │ (3) DispatchResponse
          ▼
┌──────────────────────────────────────────────┐
│  Camada de Saída (Adapter)                     │
│  - Formata resposta no schema da plataforma    │
│  - Chama API de callback da plataforma         │
└─────────┬──────────────────────────────────────┘
          │ (4) POST /reply (API da plataforma)
          ▼
┌──────────────────┐
│  Plataforma de    │ → exibida ao cliente final
│  Suporte          │
└──────────────────┘

Nota: não há broker/worker externo. Etapas (2a)-(2c) rodam na mesma task
assíncrona in-process disparada em (2). Falha do processo entre (1) e (3)
pode perder a mensagem — mitigado por processed_events (payload bruto
persistido, permite replay manual). Ver CLAUDE.md §3.2 para o racional
completo desse trade-off.

Transversal a todas as etapas:
  - correlationId propagado do (1) ao (4)
  - Logs estruturados (sem conteúdo de mensagem em texto plano)
  - Métricas: latência por etapa, taxa de escalonamento humano, taxa de erro
```

### 3.2 Camadas (Clean Architecture)

```
Domain
  ├─ Entities: Conversation, Message, Persona, KnowledgeChunk
  └─ Sem dependência externa (nenhum SDK, nenhum framework HTTP)

Application (Use Cases)
  ├─ ReceiveWebhookEvent
  ├─ ProcessIncomingMessage
  ├─ RetrieveContext        → depende de: RagPort, ClientApiPort
  ├─ GenerateResponse        → depende de: LLMProviderPort
  └─ DispatchResponse        → depende de: SupportPlatformPort

Adapters (Infra)
  ├─ inbound/
  │   └─ webhooks/mockAdapter.py   # até a plataforma real ser decidida
  ├─ outbound/
  │   ├─ llm/openRouterAdapter.py
  │   ├─ rag/pgvectorAdapter.py
  │   └─ platform/mockReplyAdapter.py   # idem
```

### 3.3 Portas (Interfaces) Centrais — MVP

```
SupportPlatformPort
  - parseIncomingEvent(rawPayload) → DomainMessage
  - sendReply(conversationId, responseText) → DeliveryResult

LLMProviderPort
  - generate(context: RetrievedContext, persona: Persona) → GeneratedResponse

RagPort
  - retrieve(query: string, tenantId: string) → KnowledgeChunk[]

# QueuePort NÃO é implementado no MVP. Mantida aqui apenas como documentação
# do ponto de extensão futuro (ver "Gatilho de reavaliação" em 2.4) — não crie
# uma implementação/adapter para ela agora.
QueuePort (reservado, não implementado no MVP)
  - enqueue(job: ProcessMessageJob) → void
  - onProcess(handler: (job) => Promise<void>) → void
```

---

## 4. Modelo de Dados (MVP — tudo no mesmo Postgres)

```
Conversation
  - id
  - tenantId              # fixo/único no MVP (single-tenant)
  - platform (mock | zendesk | blip | intercom — conforme decisão futura)
  - externalConversationId
  - status (open | escalated | closed)
  - createdAt / updatedAt

Message
  - id
  - conversationId
  - direction (inbound | outbound)
  - content            # elegível a expurgo em 90 dias — ver CLAUDE.md §3.3
  - correlationId
  - createdAt

ProcessedEvent            # deduplicação de webhook (substitui fila)
  - externalEventId (UNIQUE)
  - platform
  - receivedAt
  - rawPayload            # elegível a expurgo em 90 dias
  - status (received | processing | done | failed)

FailedMessage              # substitui DLQ no MVP
  - processedEventId (FK)
  - failedAtStep (retrieve_context | generate_response | dispatch_response)
  - errorDetail
  - retryCount
  - createdAt

KnowledgeChunk              # RAG via pgvector, mesma instância de Postgres
  - id
  - tenantId
  - content
  - embedding (vector)
  - sourceRef
```

---

## 5. Requisitos Não-Funcionais

| Requisito | Meta inicial |
|---|---|
| Latência de ACK do webhook | < 2s (p99) |
| Latência ponta a ponta (webhook → resposta enviada) | < 15s (p95) — sujeito a validação com o cliente |
| Disponibilidade do endpoint de webhook | ≥ 99.9% (best-effort no MVP, sem SLA formal) |
| Retenção de dados de conversa | 90 dias (Message, processed_events.rawPayload) |
| Multi-tenancy | Single-tenant no MVP; `tenantId` presente no schema para evolução futura |
| Taxa de escalonamento para humano | Métrica a monitorar, sem meta fixa inicial |

---

## 6. Próximos Passos (Spec Kit workflow)

1. `/constitution` — formalizar os princípios da seção 3 do `CLAUDE.md` como
   constitution do Spec Kit, se ainda não gerada.
2. `/specify` — detalhar a spec funcional do primeiro slice: webhook mock →
   RAG (pgvector) → OpenRouter → resposta mock, ponta a ponta.
3. `/plan` — a partir da decisão de stack (seção 2.4) e da arquitetura (seção
   3), gerar o plano técnico de implementação desse primeiro slice vertical.
4. `/tasks` — quebrar o plano em tarefas executáveis.
5. Quando a plataforma de suporte for escolhida, `/specify` novamente para o
   adapter real, substituindo o mock sem tocar em Domain/Application.

---

## 7. Perguntas em Aberto

- [ ] Qual plataforma de suporte será integrada primeiro? — **pendente**,
      adapter mock em uso enquanto isso.
- [x] Qual provedor de LLM é o padrão? — **OpenRouter**.
- [x] Qual vector DB / estratégia de RAG? — **pgvector**, busca vetorial direta
      (sem pipeline de retrieval híbrido no MVP).
- [x] Qual é a política de retenção de dados de conversa (LGPD)? — **90 dias**.
- [x] Multi-tenancy desde o dia 1, ou single-tenant? — **single-tenant no MVP**,
      `tenantId` já no schema.
- [ ] Qual o critério de confiança mínimo para não escalonar para humano? —
      **pendente**, definir ao especificar o use case `GenerateResponse`.
