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

**Plataforma de suporte de destino:** ainda não definida (`<TBD>`). Não implemente
nenhum adapter concreto de plataforma (Zendesk/Blip/Intercom) até que isso seja
decidido — implemente contra a interface `SupportPlatformPort` e use um adapter de
teste/mock enquanto a decisão não é tomada.

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
- Webhooks devem ser **idempotentes**. Assuma que a plataforma de origem pode
  reenviar o mesmo evento múltiplas vezes (retry de rede, timeout do lado deles).
  Use um identificador único de evento + deduplicação (tabela `processed_events`
  com constraint `UNIQUE`) antes de processar.
- Valide assinatura/HMAC do webhook (quando o provedor suportar) antes de aceitar
  qualquer payload. Rejeite silenciosamente (200 vazio ou 401, conforme o provedor)
  payloads não assinados corretamente — nunca processe "por via das dúvidas".
- Todo payload recebido deve ser persistido (`processed_events.rawPayload`) antes de
  qualquer tentativa de processamento, para permitir replay manual em caso de falha
  downstream.

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
- Deduplicação de eventos continua obrigatória: persista o `externalEventId`
  recebido no Postgres *antes* de disparar o processamento. Se o evento já
  existir, ignore silenciosamente (idempotência).
- Como não há fila com retry automático, cada etapa de processamento
  (retrieve context, chamada à LLM, envio da resposta) deve ter **seu próprio
  retry manual com backoff** (2–3 tentativas, backoff exponencial curto) e um
  tratamento explícito de falha final: registrar o erro na tabela
  `failed_messages` para permitir reprocessamento manual posterior. Isso
  substitui a DLQ neste estágio do projeto.
- Timeouts explícitos continuam obrigatórios em toda chamada de I/O externo.
- **Limite conhecido e aceito deste modelo:** se o processo cair entre o ACK do
  webhook e a conclusão do processamento em background, a mensagem pode ser
  perdida (não há persistência de fila externa garantindo entrega). Isso é um
  trade-off aceito conscientemente pelo estágio de MVP — mitigado por: (a)
  persistir o payload bruto do webhook em `processed_events` antes de processar
  (permite replay manual), e (b) manter o processo com poucas réplicas e
  reinícios controlados.
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
- Nunca logue o corpo completo de mensagens de usuário em texto plano em logs de
  produção. Logs devem conter apenas metadados (IDs de conversa, timestamps,
  status) — o conteúdo da mensagem fica no banco, com controle de acesso e
  retenção definidos.
- **Política de retenção definida para o MVP: 90 dias.** Dados em `Message` e
  `processed_events.rawPayload` devem ser elegíveis para expurgo automático após
  90 dias (job de limpeza agendado — pode ser um script manual/cron simples no
  MVP, não precisa de infraestrutura sofisticada, mas precisa existir antes do
  go-live com dados reais). Não implemente retenção indefinida "por enquanto".
- Todo dado pessoal armazenado precisa manter esse mecanismo de exclusão
  (direito ao esquecimento, LGPD Art. 18). Se uma nova tabela armazenar dado de
  conversa, ela entra automaticamente na rotina de expurgo de 90 dias, a menos
  que o usuário decida explicitamente o contrário.
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
de uma plataforma de suporte — especialmente relevante aqui porque **a
plataforma de destino ainda não foi escolhida**. Estruture o código em camadas:

```
┌─────────────────────────────────────────┐
│  Adapters (infra)                        │
│  - Webhook receivers (mock hoje;         │
│    Zendesk/Blip/Intercom quando definido)│
│  - Response senders (API clients)        │
│  - RAG client (pgvector)                 │
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
- Toda integração externa (plataforma de suporte, OpenRouter, Postgres/pgvector)
  é acessada através de uma **interface/porta** definida na camada de
  Application, implementada na camada de Adapters. Trocar de plataforma de
  suporte, ou de modelo dentro da OpenRouter, deve significar escrever/ajustar
  um adapter — nunca tocar em regra de negócio.
- Nunca vaze tipos específicos do SDK de um provedor para além da camada de
  Adapters. Converta para um modelo de domínio interno imediatamente na borda.
- Como a plataforma de suporte ainda não está definida, o adapter de webhook
  atual deve ser um **mock/stub explícito** (ex: endpoint genérico de teste),
  claramente isolado, para que a troca pelo adapter real não exija mudança na
  Application nem no Domain.

### 3.5 Observabilidade
- Toda mensagem processada deve ser rastreável ponta a ponta: um
  `correlationId` gerado na entrada do webhook e propagado por todo o
  processamento em background, RAG, chamada de LLM e resposta.
- Erros de infraestrutura (Postgres indisponível, LLM indisponível, timeout de
  RAG) devem ser distinguíveis de erros de negócio (ex: "não encontrei contexto
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
- Provedor de LLM: OpenRouter (acesso unificado a múltiplos modelos via
  uma única API/chave — flexibilidade para trocar de modelo sem trocar
  de integração)
- Vector DB / RAG: pgvector (extensão no mesmo Postgres — evita serviço extra)
- Banco de dados relacional: PostgreSQL (única instância cobre dados de
  domínio, deduplicação de eventos e vetores do RAG)
- Multi-tenancy: single-tenant no MVP (tenantId presente no schema, sem
  isolamento rígido ainda)
- Retenção de dados: 90 dias (Message, processed_events.rawPayload)
- Plataforma de suporte: <TBD — adapter mock até decisão>
- Testes: pytest + pytest-asyncio, com mocks para LLM/RAG/plataforma de suporte
- Lint/Format/Type-check: ruff (lint+format) + mypy (modo estrito)
- Deploy/Infra: single container/service em plataforma de baixo custo
  (ex: Railway, Fly.io, Render) — sem Kubernetes, sem múltiplos serviços
  no MVP
```

Racional de custo: uma única instância de Postgres (com pgvector) elimina a
necessidade de um vector DB gerenciado separado e de um broker de fila (Redis)
— reduz a arquitetura de MVP a **dois componentes pagos: o serviço web e o
banco de dados** (mais a conta de uso da OpenRouter, pay-as-you-go).

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

# Subir Postgres local (com pgvector) via docker-compose
docker compose up -d postgres

# Aplicar migrations
uv run alembic upgrade head
```

> Nota: `uv` é sugerido como gerenciador de pacotes Python (mais rápido que
> pip/poetry puro). Ajuste os comandos acima se o time preferir outra
> ferramenta antes de considerar esta seção final.

## 6. Documentos de Referência
- `SDD.md` — Software Design Document: decisão de stack, arquitetura e fluxo de dados.
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
