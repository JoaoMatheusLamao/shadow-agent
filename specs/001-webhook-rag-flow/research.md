# Research — Fluxo Ponta a Ponta (Digisac → Conhecimento → LLM → Resposta)

**Feature**: `001-webhook-rag-flow` · **Data**: 2026-07-30
**Revisão**: 2026-07-30 — **stack sem persistência** (sem Postgres); modelo OpenRouter
eleito; estudo de hospedagem.

Resolve as incógnitas técnicas do `spec.md`. Fontes sobre o Digisac: doc Postman, GitBook
oficial e libs de cliente da comunidade. Onde um campo não pôde ser 100% confirmado, está
**(confirmar em homologação)** — o formato do Digisac fica isolado no adapter.

> **Mudança de stack (decisão do usuário, 2026-07-30):** **sem banco de dados / sem
> persistência** neste estágio (Postgres seria custo desnecessário). O sistema opera
> **apenas com retries e logs**. Consequências tratadas abaixo (D3, D4, D8, D9). Isto
> **relaxa deliberadamente** o princípio de idempotência durável de `CLAUDE.md §3.1`
> (agora *best-effort* em memória) — trade-off aceito para o MVP.

---

## D1. Autenticação no Digisac
- **Decisão**: **Personal Access Token (PAT)** de longa duração, `Authorization: Bearer
  <token>`, em env `DIGISAC_API_TOKEN`. Base URL por env `DIGISAC_BASE_URL`.
- **Rationale**: mais simples que OAuth2 password grant; adequado a backend M2M.
- **Alternativa rejeitada**: OAuth2 password grant (renovação/credenciais sem ganho).

## D2. Segurança do webhook (sem HMAC nativo)
- **Decisão**: token secreto na URL — `POST /webhooks/digisac/{webhook_secret}` comparado
  a `DIGISAC_WEBHOOK_SECRET`; mismatch → `200` vazio (rejeição silenciosa). Filtro de
  borda: `event == "message.created"` e `data.isFromMe == false`.
- **Rationale**: Digisac não documenta assinatura; segredo na URL é o mecanismo possível.

## D3. Deduplicação **sem banco** (in-memory, best-effort)
- **Decisão**: cache **em memória com TTL** (ex.: `cachetools.TTLCache`, TTL ~24h) keyed
  por `data.id`. Antes de processar, checa/insere o ID; se já visto, ignora (`200`).
- **Rationale**: sem Postgres, não há `UNIQUE` durável. O retry do Digisac ocorre em
  segundos/minutos; um TTL de horas cobre a maioria enquanto o processo está vivo.
- **⚠ Limitação aceita**: o cache é **volátil** — em reinício/deploy/crash o histórico de
  dedup é perdido, podendo gerar **resposta duplicada** para uma mensagem reentregue logo
  após o restart. Mitigadores: poucos deploys, baixo volume, TTL de horas. **Gatilho de
  evolução**: se duplicidade pós-restart incomodar o negócio, reintroduzir uma store
  durável leve (Redis/DB) — não antes.
- **Alternativa rejeitada**: sem dedup — reentregas do Digisac gerariam respostas
  repetidas visíveis ao cliente (inaceitável mesmo no MVP).

## D4. Montagem do contexto da sessão — **via API do Digisac** (não mais local)
- **Decisão**: como não há tabela `messages` local, o histórico do ticket é **buscado ao
  vivo na API do Digisac** no processamento em background:
  `GET /api/v1/messages?where[contactId]=<id>` (filtrando pelo ticket aberto
  `data.ticketId`), ordenado cronologicamente, limitado a `MAX_CONTEXT_MESSAGES` (~30).
- **Rationale**: o Digisac é a fonte de verdade das mensagens; sem persistência local, é a
  única fonte do histórico. Fica no caminho de background (não no ACK), com timeout+retry.
- **Trade-off**: adiciona 1 chamada de rede por mensagem (custo de latência, coberto pela
  meta de 15s ponta a ponta). **(confirmar em homologação)** filtro exato por ticket.
- **Fallback**: se a listagem falhar após retries, usar **apenas a mensagem atual** como
  contexto e registrar em log (degrade graceful), em vez de travar.

## D5. Base de conhecimento (Excel em `docs/`) — BM25 em memória
- **Decisão**: carregar `docs/*.xlsx` no startup; recuperação **BM25** (`rank_bm25`) sobre
  coluna de texto; top-K (`KNOWLEDGE_TOP_K`=4) acima de `KNOWLEDGE_MIN_SCORE`. Leitura com
  `openpyxl`. Formato em `contracts/knowledge-excel.md`.
- **Rationale**: base pequena → zero infra extra, barato, rápido; sem embeddings/pgvector.
- **Nota**: isto já era “sem persistência” (memória) — **não muda** com a nova stack.

## D6. Geração de resposta (LLM via OpenRouter) + **modelo eleito**
- **Decisão de integração**: `httpx.AsyncClient` →
  `https://openrouter.ai/api/v1/chat/completions`; prompt = persona (`personas/*.md`) +
  histórico do ticket (D4) + trechos do conhecimento (D5); `timeout`
  (`LLM_TIMEOUT_SECONDS`=20) e `max_tokens` limitados; saída JSON
  `{ "answer": str, "can_answer": bool }` validada com pydantic. Chave em
  `OPENROUTER_API_KEY`. **Modelo é configurável** por env `OPENROUTER_MODEL`.

### Modelo eleito (resumo)
- **Default: `google/gemini-3.1-flash-lite`** (melhor custo-benefício multilíngue/PT-BR +
  latência). Fallback de qualidade: `openai/gpt-5.4-mini`. Ultra-econômico:
  `deepseek/deepseek-v4-flash`. Configurável por `OPENROUTER_MODEL`.
- **Estudo completo** (comparação, preços, racional, fontes):
  [`studies/llm-model-selection.md`](./studies/llm-model-selection.md).
- Custo OpenRouter é pay-as-you-go e, por decisão do usuário, **fora da conta de infra**.

## D7. Critério de escalonamento (fecha FR-004)
- **Decisão (MVP)**: escalonar quando **(a)** nenhum trecho do Excel acima do limiar, **ou
  (b)** LLM retorna `can_answer == false`. Escalar = transferir o ticket
  (`POST /api/v1/contacts/{contactId}/ticket/transfer` → `DIGISAC_ESCALATION_DEPARTMENT_ID`)
  e **não** enviar resposta automática. Sem tabela: o estado “escalado” vive só no log.
- **Fallback de erro** (infra ≠ negócio): falha de Digisac/LLM/Excel após retries → **log
  de erro estruturado** (não há `failed_messages`); a mensagem não é reprocessada
  automaticamente (ver D8/D9).

## D8. Resiliência: retry, backoff, timeouts — **logs no lugar de DLQ**
- **Decisão**: cada I/O externo (buscar histórico, LLM, enviar/transferir) com **retry
  2–3x + backoff exponencial curto** (`tenacity`) e **timeout explícito**. Processamento
  disparado por `BackgroundTasks` após o ACK. **Falha final → log estruturado** (nível
  ERROR) com `correlationId`, `failedAtStep` e detalhe do erro (sem PII).
- **Rationale**: substitui a fila/DLQ e a tabela `failed_messages` do desenho anterior; sem
  persistência, o **log é o sistema de registro** para diagnóstico/replay manual.
- **⚠ Limitação aceita**: sem tabela de falhas nem payload persistido, **não há replay
  automático**; o replay depende de reprocessar manualmente a partir do log/Digisac. Se o
  processo cair no meio do background, a mensagem se perde (Digisac pode reentregar).

## D9. **Sem persistência** — estado efêmero + LGPD simplificada
- **Decisão**: **nenhum banco**. Estado é transitório em memória durante o processamento;
  nada de conversa é gravado em repouso. Sem SQLAlchemy/Alembic/asyncpg.
- **LGPD**: como **não há PII em repouso**, o job de expurgo de 90 dias torna-se
  **desnecessário**. Regra que permanece: **logs sem conteúdo de mensagem** (só
  IDs/metadados/status) — o conteúdo trafega em memória e vai ao Digisac, não a um store
  nosso. Isso **reduz a superfície LGPD** do serviço.
- **Rationale**: elimina o segundo componente pago (banco), atendendo ao pedido de custo.

## D10. Observabilidade — **logs como sistema de registro**
- **Decisão**: `correlationId` (UUID) na borda do webhook, propagado a todas as etapas e
  presente em **todo log estruturado** (`structlog`/JSON). Logs **sem conteúdo**; distinguir
  erro de infra de escalonamento de negócio. Como não há banco, a rastreabilidade ponta a
  ponta (FR-008/SC-004) é feita **via agregação de logs** (buscar pelo `correlationId`).
- **Implicação de hospedagem**: preferir plataforma com **retenção/consulta de logs**
  integrada (ver D11).

## D11. Hospedagem e custo mensal (resumo)
Alvo: **1 container stateless sempre-ligado** (sem banco). **Restrição decisiva**: o
processamento roda em **background após o ACK 202**, logo a instância **precisa continuar
viva** para concluir a chamada à LLM ⇒ **evitar serverless scale-to-zero** (Cloud Run/
Lambda; cold start ainda ameaça o ACK < 2s).

- **Eleito: DigitalOcean App Platform (~US$5/mês, só infra)**; alternativa mais barata
  **Hetzner CX23 (~US$4,3)**. Sem banco ⇒ **um único componente pago**. OpenRouter e
  Digisac **não entram** na conta de infra (decisão do usuário).
- **Estudo completo** (comparação de AWS/GCP/DO/Fly/Railway/Render/Hetzner, racional
  técnico e fontes): [`studies/hosting-cost-study.md`](./studies/hosting-cost-study.md).

## D12. Ferramentas de qualidade e testes
- **Decisão**: `ruff` (lint+format), `mypy` (estrito), `pytest`+`pytest-asyncio`. Testes de
  unidade com **fakes das portas** (Digisac, LLM, conhecimento), sem rede real. Cobertura:
  use cases da Application, **dedup in-memory**, retry/log de falha. **Removidas** deps de
  banco (SQLAlchemy/Alembic/asyncpg).

---

### Incógnitas remanescentes (não bloqueiam; confirmar em homologação)
1. Campos exatos do payload (`isFromMe`/`type`) e do `GET /api/v1/messages` (filtro por
   ticket) — isolados no adapter.
2. Body exato de `POST /api/v1/messages` e do transfer de ticket.
3. Slug exato do modelo no OpenRouter (`google/gemini-3.1-flash-lite` a confirmar).
