# CLAUDE.md — Contexto Operacional do Projeto

> Este arquivo é a fonte de verdade para qualquer sessão do Claude Code neste repositório.
> Leia-o integralmente antes de propor qualquer plano de implementação.

## 1. Identidade e Escopo do Projeto

Este projeto é um **middleware/oráculo de IA para suporte ao cliente**. Ele NÃO é a
plataforma de atendimento (Zendesk, Blip, Intercom) — ele vive "atrás" dela, como um
serviço backend desacoplado.

Fluxo de responsabilidade:
1. Receber eventos de mensagem via **Webhook** de uma plataforma de suporte de terceiros.
2. Enriquecer o contexto consultando uma **base de conhecimento (RAG)** e/ou **APIs
   internas do cliente** (CRM, pedidos, faturas, etc).
3. Processar a melhor resposta usando uma **LLM com Persona definida** (tom, limites,
   escopo de atuação, política de escalonamento para humano).
4. Devolver a resposta para a plataforma de origem via **API de callback**.

Este serviço é **plataforma-agnóstico por design**. Nunca assuma que o adapter de
entrada será o único. Toda integração de fornecedor deve ser isolada atrás de uma
interface/porta — ver seção 4 (Clean Architecture).

**Estágio do projeto: MVP.** Prioridade explícita: velocidade de entrega e baixo
custo operacional. Isso não relaxa segurança/LGPD nem a qualidade de código — mas
relaxa deliberadamente a robustez de infraestrutura (ver seção 3.2).

**Plataforma de suporte de destino:** **Digisac** (definida em 2026-07-30). A
entrada é um webhook do Digisac e a saída é a API do Digisac (auth OAuth2 Bearer).
Implemente o adapter concreto do Digisac **contra a interface `SupportPlatformPort`**
— nunca vaze o formato específico do Digisac para Domain/Application, para que
adicionar outra plataforma no futuro não exija tocar na lógica de negócio.

**Base de conhecimento:** um **arquivo Excel na pasta `docs/`**, consultado
diretamente no MVP (base pequena, **sem banco vetorial/pgvector** — ver §4). A
adoção de pgvector/busca semântica é evolução pós-MVP, não parte do escopo atual.

## 2. Papel do Claude Code Neste Projeto

Você (Claude Code) é o **desenvolvedor deste sistema**, não o agente de IA que ele
expõe em produção. Não confunda os dois níveis:
- O "agente de suporte" é o software que está sendo construído.
- Você está construindo a infraestrutura, os adapters, a orquestração e os testes
  que sustentam esse agente.

Não implemente a persona/prompt de produção dentro da lógica de infraestrutura.
Prompts de sistema, políticas de persona e few-shots devem ficar isolados em
arquivos de configuração versionados (ex: `personas/*.md` ou `prompts/*.yaml`),
nunca hardcoded em código de aplicação.

## 3. Princípios de Desenvolvimento — Não Negociáveis

Este é um sistema **de produção (MVP)**, não um protótipo descartável. Trate cada
requisito abaixo como obrigatório, mesmo que o pedido do usuário não o mencione
explicitamente.

### 3.1 Resiliência de Webhooks
- Todo endpoint de webhook deve responder **imediatamente** (< 2s) com um ACK e
  processar o payload de forma assíncrona. Nunca bloqueie a resposta HTTP esperando
  a conclusão da chamada à LLM ou ao RAG.
- Webhooks devem ser **idempotentes**. Assuma que a plataforma de origem (Digisac)
  pode reenviar o mesmo evento múltiplas vezes (retry de rede, timeout do lado deles).
  Use um identificador único de evento + deduplicação antes de processar.
  **Decisão de MVP (2026-07-30 — sem banco):** a deduplicação é *best-effort* via
  **cache em memória com TTL** (chave = `data.id`), **não** uma constraint `UNIQUE`
  durável. Limitação aceita: em reinício/crash o cache zera, podendo gerar resposta
  duplicada para uma reentrega logo após restart. Gatilho de reversão: se isso
  incomodar o negócio, introduzir store durável leve (Redis/DB) — não antes.
- Valide o webhook antes de aceitar: o Digisac **não** expõe HMAC, então use um
  **token secreto na URL** (`/webhooks/digisac/{secret}`) + filtro de evento
  (`message.created`, `isFromMe=false`). Rejeite silenciosamente (200 vazio) o que
  não casar — nunca processe "por via das dúvidas".
- **Decisão de MVP (sem banco):** o payload **não** é persistido (não há
  `processed_events`). Não há replay durável — o registro para diagnóstico é o **log
  estruturado** (ver §3.2/§3.5). Se a plataforma reentregar, o fluxo reprocessa.

### 3.2 Processamento Assíncrono — Sem Fila Dedicada (Decisão de MVP)

Este projeto é um MVP com restrição explícita de custo e velocidade de entrega:
**não há fila dedicada** (sem Redis/Celery/BullMQ/SQS). Isso é uma decisão
deliberada, não um atalho — trate-a como tal e não reintroduza infraestrutura de
fila "por precaução".

Ainda assim, os princípios de resiliência não são opcionais:
- O handler do webhook responde **imediatamente** (200/202) e dispara o
  processamento em background **dentro do mesmo processo** via `BackgroundTasks`
  do FastAPI. Nunca faça a chamada à LLM/RAG no caminho síncrono da resposta HTTP
  ao webhook.
- Deduplicação de eventos continua obrigatória, porém **sem banco (decisão de MVP,
  2026-07-30):** cheque/marque o `externalEventId` (= `data.id`) num **cache em
  memória com TTL** *antes* de disparar o processamento. Se já visto, ignore
  silenciosamente (idempotência *best-effort*, volátil — ver §3.1).
- Como não há fila com retry automático, cada etapa de processamento (buscar
  histórico do ticket, chamada à LLM, envio da resposta) deve ter **seu próprio
  retry manual com backoff** (2–3 tentativas, backoff exponencial curto) e um
  tratamento explícito de falha final: **registrar o erro em log estruturado**
  (etapa + tipo de erro + `correlationId`). **Não há tabela `failed_messages`**
  (sem banco) — o log é o registro para diagnóstico/replay manual. Isso substitui
  a DLQ neste estágio.
- Timeouts explícitos continuam obrigatórios em toda chamada de I/O externo.
- **Limite conhecido e aceito deste modelo:** sem fila **e sem banco**, se o
  processo cair entre o ACK do webhook e a conclusão do background, a mensagem pode
  ser perdida. Trade-off aceito conscientemente no MVP — mitigado por: (a) o Digisac
  pode reentregar o evento, e (b) manter o processo com poucas réplicas e reinícios
  controlados. (Não há mais payload persistido para replay — ver §3.1.)
- Reavaliar a introdução de uma fila real (Celery+Redis, ou similar) é um
  gatilho explícito de evolução pós-MVP quando: volume de mensagens crescer a
  ponto de degradar a latência do processo principal, ou a perda ocasional de
  mensagem em crash deixar de ser aceitável para o negócio. Não implemente essa
  evolução preventivamente.
- Se um pedido do usuário implicar reintroduzir fila, worker separado, ou
  infraestrutura adicional (Redis, RabbitMQ, etc.) sem que o gatilho de evolução
  acima tenha sido explicitamente confirmado pelo usuário, pare e pergunte antes
  de implementar — isso contradiz a decisão de MVP registrada aqui e em `SDD.md`.

### 3.3 Segurança e LGPD
- Este sistema processa dados de conversas de clientes reais. Trate todo payload
  como **potencialmente sensível** (PII: nome, CPF, e-mail, telefone, dados
  financeiros mencionados em texto livre).
- Nunca logue o corpo de mensagens de usuário em logs de produção. Logs contêm
  apenas metadados (IDs de conversa/mensagem, timestamps, status, `correlationId`).
- **Decisão de MVP (2026-07-30 — sem banco): não há PII em repouso.** O serviço
  **não persiste** conteúdo de conversa (o conteúdo trafega em memória e vai ao
  Digisac). Consequência: a política de retenção de 90 dias **não exige job de
  expurgo** neste estágio — não há dado de conversa armazenado para expurgar.
- **Gatilho de reavaliação:** se no futuro o serviço voltar a persistir conteúdo de
  conversa (ex.: reintroduzir banco/Redis para dedup durável ou histórico), a
  retenção de 90 dias + expurgo (direito ao esquecimento, LGPD Art. 18) volta a ser
  obrigatória para qualquer store que guarde dado de conversa.
- Segredos (API key da OpenRouter, credenciais de webhook, tokens da plataforma
  de suporte) NUNCA em código, `.env` commitado, ou logs. Sempre via variáveis
  de ambiente/secret manager, com `.env.example` documentando as chaves
  necessárias sem valores reais.
- Toda chamada de saída para a API da plataforma de suporte (resposta ao
  cliente) deve passar por uma camada de sanitização/validação — nunca envie a
  saída bruta da LLM sem checagem de formato esperado pelo provedor.
- **MVP é single-tenant** (ver seção 4): não há isolamento de múltiplos clientes
  em produção ainda. Mesmo assim, inclua `tenantId` no schema desde já (valor
  fixo/único no MVP) para não exigir migração de dados quando o produto evoluir
  para multi-tenant.

### 3.4 Clean Architecture — Isolamento de Plataforma
A dependência mais perigosa deste sistema é o acoplamento ao formato específico
de uma plataforma de suporte. A plataforma atual é o **Digisac**, mas mantenha-a
isolada atrás de `SupportPlatformPort` para permitir trocar/adicionar plataformas
sem tocar em regra de negócio. Estruture o código em camadas:

```
┌─────────────────────────────────────────┐
│  Adapters (infra)                        │
│  - Webhook receivers (Digisac)           │
│  - Response senders (API do Digisac)     │
│  - Knowledge client (Excel em docs/;     │
│    pgvector adiado p/ pós-MVP)           │
│  - LLM client (OpenRouter)               │
├─────────────────────────────────────────┤
│  Application (use cases)                 │
│  - ProcessIncomingMessage                │
│  - RetrieveContext                       │
│  - GenerateResponse                      │
│  - DispatchResponse                      │
├─────────────────────────────────────────┤
│  Domain (regras de negócio puras)        │
│  - Conversation, Message, Persona        │
│  - Sem dependência de SDK de LLM,        │
│    de HTTP framework, ou de driver de BD │
└─────────────────────────────────────────┘
```

Regras rígidas:
- A camada de **Domain** não importa nada de bibliotecas de infraestrutura (SDK
  de LLM, cliente HTTP, driver de banco, SDK do provedor de suporte).
- Toda integração externa (plataforma de suporte Digisac, OpenRouter, base de
  conhecimento no Excel) é acessada através de uma **interface/porta**
  definida na camada de
  Application, implementada na camada de Adapters. Trocar de plataforma de
  suporte, ou de modelo dentro da OpenRouter, deve significar escrever/ajustar
  um adapter — nunca tocar em regra de negócio.
- Nunca vaze tipos específicos do SDK de um provedor para além da camada de
  Adapters. Converta para um modelo de domínio interno imediatamente na borda.
- A plataforma de suporte é o **Digisac**: o adapter de webhook (entrada) e o
  adapter de resposta (saída) são implementações concretas do Digisac, **isoladas
  atrás de `SupportPlatformPort`**. Domain e Application não conhecem o formato do
  Digisac — adicionar outra plataforma no futuro é escrever outro adapter, sem
  tocar em regra de negócio. Em testes, use um adapter fake da porta (não a API
  real do Digisac).

### 3.5 Observabilidade
- Toda mensagem processada deve ser rastreável ponta a ponta: um
  `correlationId` gerado na entrada do webhook e propagado por todo o
  processamento em background, RAG, chamada de LLM e resposta.
- Erros de infraestrutura (Digisac indisponível, LLM indisponível, falha ao ler o
  Excel) devem ser distinguíveis de erros de negócio (ex: "não encontrei contexto
  suficiente para responder") tanto em logs quanto em métricas.
- Nunca falhe silenciosamente. Se uma etapa não pode ser concluída, o sistema
  deve ter um caminho de fallback definido (ex: marcar para escalonamento
  humano) — nunca deixar a conversa "no vácuo".

### 3.6 Qualidade de Código Esperada
- Cobertura de testes obrigatória para: use cases da camada Application, lógica
  de deduplicação de webhook, e lógica de retry manual/registro em
  `failed_messages`. Testes de unidade não devem depender de rede real (mocks
  para LLM/OpenRouter, RAG e plataforma de suporte).
- Ao propor uma implementação, sempre considere e declare explicitamente: o que
  acontece se a LLM (via OpenRouter) der timeout ou erro, o que acontece se o
  RAG retornar vazio, o que acontece se o webhook chegar duplicado, o que
  acontece se o processo cair no meio do processamento em background.
- Não adicione abstrações especulativas (ex: suporte a multi-tenant real, ou a
  fila dedicada) a menos que solicitado. A Clean Architecture já garante a
  extensibilidade — não é necessário construí-la preventivamente em código.
- Commits e PRs devem descrever o *porquê*, não o *o quê* (o diff já mostra o
  quê).

## 4. Stack Técnica

> **STATUS: DEFINIDO (MVP).** Ver `SDD.md`, seção "Decisão de Stack", para a
> justificativa completa.

```
- Linguagem/Runtime: Python 3.12+
- Framework HTTP: FastAPI (ASGI, via Uvicorn)
- Processamento assíncrono: in-process via BackgroundTasks do FastAPI
  (sem fila/worker externo — ver seção 3.2)
- Provedor de LLM: OpenRouter (acesso unificado a múltiplos modelos via uma única
  API/chave). Modelo default: **`google/gemini-3.1-flash-lite`** (melhor
  custo-benefício multilíngue/PT-BR + latência), configurável por env
  `OPENROUTER_MODEL`; fallback de qualidade `openai/gpt-5.4-mini`; opção
  ultra-econômica `deepseek/deepseek-v4-flash`. Ver `SDD.md` e research do plano.
- Base de conhecimento (MVP): arquivo Excel em `docs/`, carregado em memória e
  consultado por busca lexical (BM25). Sem banco vetorial. pgvector: adiado pós-MVP.
- **Persistência: NENHUMA (decisão de MVP, 2026-07-30 — custo).** Sem banco de
  dados. Estado transitório em memória; **dedup por cache em memória com TTL**
  (volátil); falhas e rastreamento em **logs estruturados**. Sem SQLAlchemy/Alembic/
  asyncpg. (Postgres/pgvector: reavaliar só se dedup durável/histórico persistido/
  busca semântica se tornarem necessários — ver §3.1/§3.2 e SDD §7.)
- Histórico da conversa: buscado ao vivo na API do Digisac (sem base local).
- Multi-tenancy: single-tenant no MVP (`tenantId` nos objetos, sem isolamento rígido).
- Retenção de dados: **N/A no MVP** — sem PII em repouso (nada persistido). Ver §3.3.
- Plataforma de suporte: Digisac (webhook de entrada + API de saída; **Personal
  Access Token** Bearer).
- Testes: pytest + pytest-asyncio, com fakes das portas (LLM/conhecimento/Digisac).
- Lint/Format/Type-check: ruff (lint+format) + mypy (modo estrito).
- Deploy/Infra: **container único sempre-ligado** (o background pós-ACK exige
  instância viva ⇒ evitar serverless scale-to-zero). **Plataforma eleita:
  DigitalOcean App Platform (~US$5/mês)**; alternativa mais barata Hetzner CX23
  (~US$4,3). Sem Kubernetes, sem múltiplos serviços.
```

Racional de custo: **sem banco** no MVP, a arquitetura fica com **um único
componente pago: o serviço web** (~US$5/mês em DigitalOcean App Platform, só infra),
mais a conta de uso da OpenRouter (pay-as-you-go, à parte). O Digisac também não
entra na conta de infra. Dedup em memória e histórico via API do Digisac eliminam a
necessidade de qualquer store gerenciado neste estágio.

## 5. Comandos do Projeto

```bash
# Instalação de dependências
uv sync

# Rodar em desenvolvimento
uv run uvicorn app.main:app --reload

# Rodar suíte de testes
uv run pytest

# Lint
uv run ruff check .

# Format
uv run ruff format .

# Type-check
uv run mypy .
```

> Sem banco no MVP: não há `docker compose up postgres` nem `alembic upgrade` — o
> serviço sobe sozinho (`uvicorn`), sem dependência de infraestrutura de dados.

> Nota: `uv` é sugerido como gerenciador de pacotes Python (mais rápido que
> pip/poetry puro). Ajuste os comandos acima se o time preferir outra
> ferramenta antes de considerar esta seção final.

## 6. Documentos de Referência
- `SDD.md` — Software Design Document: decisão de stack, arquitetura e fluxo de dados.
- `specs/001-webhook-rag-flow/design/flow-sequence.md` — **diagrama de sequência (mermaid) do
  fluxo escopado; artefato vivo, mantenha SEMPRE atualizado** a cada mudança de
  comportamento no fluxo (spec/plan/implementação), na mesma alteração.
- `specs/001-webhook-rag-flow/` — spec, plan, research, data-model, contracts, quickstart.
- `.claudeignore` — arquivos/pastas fora do contexto de análise.
- `personas/` — definições de persona da LLM em produção (quando existirem).
- `.specify/` — workflow do Spec Kit (constitution/specify/plan/tasks).

## 7. Comportamento Esperado do Claude Code
- Antes de gerar código para uma nova funcionalidade, identifique explicitamente
  em qual camada (Domain / Application / Adapters) ela pertence.
- Se um pedido do usuário implicar violar um dos princípios da seção 3 (ex:
  "loga a mensagem inteira para debug", "guarda os dados para sempre", "adiciona
  uma fila"), sinalize o risco/a contradição com a decisão de MVP antes de
  implementar e sugira a alternativa combinada — não implemente silenciosamente.
- A plataforma de suporte e o modelo específico via OpenRouter ainda podem ser
  ajustados — trate ambos como configuráveis (variável de ambiente / adapter
  plugável), nunca como constante hardcoded.
