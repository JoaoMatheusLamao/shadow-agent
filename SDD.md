# SDD.md — Software Design Document
**Projeto:** Middleware/Oráculo de IA para Suporte ao Cliente
**Status:** Stack e decisões-chave de MVP definidas — plataforma de suporte definida (Digisac)
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

**Decisão final:** Python 3.12 + FastAPI + BackgroundTasks (sem fila dedicada) +
OpenRouter como provedor de LLM. **Sem banco de dados no MVP.**
**Data:** 2026-07-19 (revisado 2026-07-30)
**Responsável:** usuário do projeto

> **Atualização 2026-07-30 (a):** plataforma definida como **Digisac**; base de
> conhecimento passou a ser um **Excel em `docs/`** consultado diretamente; pgvector
> **adiado para pós-MVP**.
>
> **Atualização 2026-07-30 (b) — SEM PERSISTÊNCIA:** por decisão de custo, o MVP
> **não usa banco de dados**. Consequências (substituem as menções a Postgres/tabelas
> nas seções 3 e 4): (1) **deduplicação** via **cache em memória com TTL** (volátil,
> *best-effort* — não `UNIQUE` durável); (2) **histórico do ticket** buscado ao vivo
> na **API do Digisac** (não há tabela `messages`); (3) **falhas** em **logs
> estruturados** (não há `failed_messages`); (4) **sem PII em repouso** ⇒ retenção de
> 90 dias sem job de expurgo. O sistema opera **apenas com retries e logs**.
>
> **Atualização 2026-07-30 (c) — modelo e hospedagem:** modelo LLM default
> **`google/gemini-3.1-flash-lite`** (melhor custo-benefício PT-BR + latência),
> configurável; fallback `openai/gpt-5.4-mini`; ultra-econômico
> `deepseek/deepseek-v4-flash`. Hospedagem eleita **DigitalOcean App Platform
> (~US$5/mês, só infra)**; alternativa Hetzner CX23 (~US$4,3). Ver `research.md` do
> plano (D6, D11).

**Gatilho de reavaliação:** se o volume de mensagens ou os requisitos de
confiabilidade de entrega crescerem a ponto de a perda ocasional de mensagem em
crash do processo deixar de ser aceitável, reavaliar introdução de
Celery+Redis (mesma linguagem, sem necessidade de reescrever a lógica de
negócio, dado o isolamento via `QueuePort` já previsto na Clean Architecture —
ver seção 3.3).

---

## 3. Arquitetura de Alto Nível

### 3.1 Fluxo de Dados (MVP — sem fila dedicada)

> **⚠ Diagrama supersedido.** O ASCII abaixo reflete o desenho anterior (com
> Postgres: `processed_events`, `failed_messages`). No MVP vigente **não há banco**
> (Atualização 2026-07-30 (b)): dedup em cache de memória, histórico via API do
> Digisac, falhas em log. **Fluxo canônico e atualizado:**
> `specs/001-webhook-rag-flow/design/flow-sequence.md` (diagrama mermaid, artefato vivo).

```
┌──────────────────┐
│  Plataforma de    │   (Digisac)
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
│    │  Histórico do ticket aberto (Digisac)│    │
│    │  + base de conhecimento (Excel/docs) │    │
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
  ├─ Entities: Conversation, Message, Persona, KnowledgeEntry
  └─ Sem dependência externa (nenhum SDK, nenhum framework HTTP)

Application (Use Cases)
  ├─ ReceiveWebhookEvent
  ├─ ProcessIncomingMessage
  ├─ RetrieveContext        → depende de: KnowledgePort, ClientApiPort
  ├─ GenerateResponse        → depende de: LLMProviderPort
  └─ DispatchResponse        → depende de: SupportPlatformPort

Adapters (Infra)
  ├─ inbound/
  │   └─ webhooks/digisacAdapter.py   # recebe o webhook do Digisac
  ├─ outbound/
  │   ├─ llm/openRouterAdapter.py
  │   ├─ knowledge/excelAdapter.py    # lê a base de conhecimento (docs/*.xlsx)
  │   │                               # pgvectorAdapter: adiado p/ pós-MVP
  │   └─ platform/digisacReplyAdapter.py   # envia a resposta via API do Digisac
```

### 3.3 Portas (Interfaces) Centrais — MVP

```
SupportPlatformPort
  - parseIncomingEvent(rawPayload) → DomainMessage
  - sendReply(conversationId, responseText) → DeliveryResult

LLMProviderPort
  - generate(context: RetrievedContext, persona: Persona) → GeneratedResponse

KnowledgePort              # no MVP: implementado pelo excelAdapter (docs/*.xlsx)
  - retrieve(query: string, tenantId: string) → KnowledgeEntry[]

# QueuePort NÃO é implementado no MVP. Mantida aqui apenas como documentação
# do ponto de extensão futuro (ver "Gatilho de reavaliação" em 2.4) — não crie
# uma implementação/adapter para ela agora.
QueuePort (reservado, não implementado no MVP)
  - enqueue(job: ProcessMessageJob) → void
  - onProcess(handler: (job) => Promise<void>) → void
```

---

## 4. Modelo de Dados

> **⚠ SUPERSEDIDO pela Atualização 2026-07-30 (b): SEM BANCO no MVP.** As tabelas
> abaixo descrevem o desenho **anterior** com Postgres e são mantidas apenas como
> referência histórica / desenho de evolução (quando/se a persistência voltar). No
> MVP atual **nada disto é persistido**: `Conversation`/`Message` são objetos
> transitórios em memória; `ProcessedEvent` vira **cache de dedup em memória (TTL)**;
> `FailedMessage` vira **evento de log estruturado**; `KnowledgeEntry` é lido do
> Excel. Ver `specs/001-webhook-rag-flow/data-model.md` para o modelo vigente.

```
Conversation
  - id
  - tenantId              # fixo/único no MVP (single-tenant)
  - platform (digisac — extensível a outras plataformas no futuro)
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

KnowledgeEntry              # base de conhecimento no MVP: lida do Excel (docs/)
  - id                       # não persistido em Postgres no MVP; carregado do
  - tenantId                 # arquivo Excel. Sem coluna embedding enquanto pgvector
  - content                  # estiver adiado (ver gatilho de evolução na seção 7).
  - sourceRef                # (ex.: aba/linha do Excel de origem)
```

---

## 5. Requisitos Não-Funcionais

| Requisito | Meta inicial |
|---|---|
| Latência de ACK do webhook | < 2s (p99) |
| Latência ponta a ponta (webhook → resposta enviada) | < 15s (p95) — sujeito a validação com o cliente |
| Disponibilidade do endpoint de webhook | ≥ 99.9% (best-effort no MVP, sem SLA formal) |
| Retenção de dados de conversa | **N/A no MVP** — sem persistência / sem PII em repouso (nada armazenado); regra retorna se a persistência voltar |
| Custo de infra (nosso projeto) | ~US$5/mês (DigitalOcean App Platform) — OpenRouter/Digisac fora da conta |
| Multi-tenancy | Single-tenant no MVP; `tenantId` presente nos objetos para evolução futura |
| Taxa de escalonamento para humano | Métrica a monitorar, sem meta fixa inicial |

---

## 6. Próximos Passos (Spec Kit workflow)

1. `/constitution` — formalizar os princípios da seção 3 do `CLAUDE.md` como
   constitution do Spec Kit, se ainda não gerada.
2. `/specify` — detalhar a spec funcional do primeiro slice: webhook do Digisac →
   contexto do ticket + conhecimento (Excel/docs) → OpenRouter → resposta enviada
   ao Digisac, ponta a ponta. **(feito — ver `specs/001-webhook-rag-flow`)**
3. `/plan` — a partir da decisão de stack (seção 2.4) e da arquitetura (seção
   3), gerar o plano técnico de implementação desse primeiro slice vertical.
4. `/tasks` — quebrar o plano em tarefas executáveis.
5. Plataforma escolhida (Digisac): o adapter de entrada/saída é implementado
   contra `SupportPlatformPort`, sem tocar em Domain/Application. Se a base de
   conhecimento crescer, `/specify` para a evolução com pgvector/busca semântica.

---

## 7. Perguntas em Aberto

- [x] Qual plataforma de suporte será integrada primeiro? — **Digisac**
      (definido em 2026-07-30). Entrada via webhook do Digisac, saída via API do
      Digisac (OAuth2 Bearer). Detalhes de payload/endpoint a confirmar no `/plan`.
- [x] Qual provedor de LLM é o padrão? — **OpenRouter**.
- [x] Qual vector DB / estratégia de RAG? — **pgvector adiado para pós-MVP**. No
      MVP a base de conhecimento é um **arquivo Excel em `docs/`**, consultado
      diretamente (base pequena, sem banco vetorial). pgvector/busca semântica
      vira gatilho de evolução quando a base crescer ou a busca direta deixar de
      ser suficiente.
- [x] Usar banco de dados no MVP? — **NÃO** (decisão 2026-07-30, custo). Sem
      persistência: dedup em cache de memória (TTL, volátil), histórico via API do
      Digisac, falhas em logs. Reintroduzir store durável (Redis/Postgres) é gatilho
      de evolução se dedup pós-restart/perda em crash deixarem de ser aceitáveis.
- [x] Qual é a política de retenção de dados de conversa (LGPD)? — **N/A no MVP**:
      sem PII em repouso (nada persistido). Volta a valer (90 dias + expurgo) se a
      persistência de conteúdo for reintroduzida.
- [x] Multi-tenancy desde o dia 1, ou single-tenant? — **single-tenant no MVP**,
      `tenantId` já no schema.
- [~] Qual o critério de confiança mínimo para não escalonar para humano? —
      **default definido no spec 001** (escalonar quando não há conteúdo relevante
      no Excel ou a LLM sinaliza incerteza); critério numérico fino a detalhar no
      `/plan` ao desenhar `GenerateResponse`.
- [x] Onde hospedar (custo mensal)? — **DigitalOcean App Platform** (~US$5/mês, só
      infra), container único sempre-ligado (o background pós-ACK exige instância
      viva ⇒ evitar scale-to-zero). Alternativa mais barata: Hetzner CX23 (~US$4,3).
      Evitar Cloud Run/Lambda neste padrão. Ver `research.md` D11.
- [x] Qual modelo no OpenRouter? — default **`google/gemini-3.1-flash-lite`**
      (custo-benefício PT-BR + latência), configurável; fallback
      `openai/gpt-5.4-mini`; ultra-econômico `deepseek/deepseek-v4-flash`. Ver D6.
