# Implementation Plan: Fluxo Ponta a Ponta — Digisac → Conhecimento → LLM → Resposta

**Branch**: `001-webhook-rag-flow` | **Date**: 2026-07-30 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `specs/001-webhook-rag-flow/spec.md`

## Summary

Serviço backend (middleware/oráculo) que recebe o **webhook do Digisac** quando um cliente
envia mensagem, faz **dedup best-effort (cache em memória)**, ACK imediato (< 2s) e processa
em **background in-process**: **busca o histórico do ticket na API do Digisac**, consulta a
**base de conhecimento (Excel em `docs/`, BM25)**, gera resposta com **LLM via OpenRouter
(`google/gemini-3.1-flash-lite`) + persona**, e devolve pela **API do Digisac** — ou
**escala** para humano quando falta contexto. **Sem banco/persistência**: resiliência por
**retry + backoff + logs estruturados** (sem `failed_messages`). Rastreabilidade por
`correlationId` (via logs). **Sem PII em repouso** ⇒ LGPD simplificada. Single-tenant.
Decisões em [research.md](./research.md); fluxo canônico em [design/flow-sequence.md](./design/flow-sequence.md).

## Technical Context

**Language/Version**: Python 3.12+

**Primary Dependencies**: FastAPI + Uvicorn (ASGI); `httpx` (async — OpenRouter/Digisac);
`openpyxl` + `rank_bm25` (conhecimento); `pydantic`/`pydantic-settings`; `tenacity` (retry);
`cachetools` (dedup TTL in-memory); `structlog` (logs). `BackgroundTasks` do FastAPI.
**Sem** SQLAlchemy/Alembic/asyncpg (sem banco).

**Storage**: **Nenhum.** Sem banco de dados. Estado transitório em memória; dedup em cache
volátil; falhas/rastreamento em logs. (pgvector/Postgres: adiados, ver SDD §7.)

**Testing**: pytest + pytest-asyncio; fakes das portas (Digisac/LLM/conhecimento), sem rede
real.

**Target Platform**: container único **sempre-ligado** (background pós-ACK exige instância
viva ⇒ evitar scale-to-zero). **Hosting eleito: DigitalOcean App Platform (~US$5/mês, só
infra)**; alternativa mais barata Hetzner CX23 (~US$4,3). Ver research.md D11.

**Project Type**: web service (backend único, Clean Architecture).

**Performance Goals**: ACK do webhook < 2s (p99); ponta a ponta < 15s (p95) — SC-001.

**Constraints**: sem fila e **sem banco**; timeouts explícitos em todo I/O; sem PII em log;
sanitizar saída da LLM antes de enviar; dedup best-effort (volátil).

**Scale/Scope**: volume baixo/moderado; single-tenant; base de conhecimento pequena (Excel).

## Constitution Check

*Constitution formal ainda é template; gates derivados de `CLAUDE.md §3`.*

| Princípio (CLAUDE.md) | Como o plano atende | Status |
|---|---|---|
| §3.1 Resiliência de webhook | ACK 202 < 2s; validação antes de processar; **dedup best-effort in-memory** (não durável) | ⚠ parcial — ver Desvio 1 |
| §3.2 Assíncrono sem fila | `BackgroundTasks`; retry + backoff; **falhas em log** (sem `failed_messages`) | ⚠ parcial — ver Desvio 2 |
| §3.3 Segurança/LGPD | secrets em env; **sem PII em repouso** (nada persistido); logs sem conteúdo; sanitização da saída | ✅ (LGPD até simplificada) |
| §3.4 Clean Architecture | Domain puro; portas na Application; Digisac/OpenRouter/Excel só em Adapters | ✅ |
| §3.5 Observabilidade | `correlationId` ponta a ponta via logs; infra ≠ negócio; fallback = escalonar | ✅ |
| §3.6 Qualidade/testes | fakes de porta; cobertura de use cases, dedup, retry/log; ruff+mypy | ✅ |

### Desvios justificados (decisão explícita do usuário — custo)

| # | Desvio | Motivo | Mitigação / Gatilho de reversão |
|---|---|---|---|
| 1 | Idempotência **best-effort** (cache volátil) em vez de `UNIQUE` durável (§3.1) | Sem banco (custo) | TTL de horas + baixo volume; se duplicidade pós-restart incomodar → store durável leve (Redis) |
| 2 | Falhas em **log** em vez de `failed_messages`/replay durável (§3.2) | Sem banco (custo) | Logs estruturados por `correlationId`; Digisac reentrega; se perda for inaceitável → reintroduzir persistência |

**Gate: PASS com desvios registrados.** Os desvios foram **escolhidos explicitamente pelo
usuário** (remover o banco para reduzir custo) e reconciliados em `CLAUDE.md`/`SDD.md`. Não
reintroduzir banco/fila sem o gatilho acima ser confirmado (CLAUDE.md §3.2).

## Project Structure

### Documentation (this feature)

```text
specs/001-webhook-rag-flow/
├── spec.md · plan.md · research.md · data-model.md · quickstart.md   # speckit (raiz)
├── contracts/{webhook-inbound,digisac-outbound,ports,knowledge-excel}.md   # speckit
├── checklists/requirements.md                                        # speckit
├── tasks.md            # (gerado por /speckit-tasks) — speckit
├── design/flow-sequence.md            # diagrama mermaid (artefato vivo)
├── studies/{hosting-cost-study,llm-model-selection}.md               # pesquisas
└── tbd/{product,technical}.md          # itens em aberto (PO / técnico)
```

### Source Code (repository root)

```text
app/
├── main.py                       # FastAPI app + wiring (DI) + BackgroundTasks
├── config.py                     # pydantic-settings (env)
├── domain/
│   └── entities.py               # DomainMessage, Conversation, Persona, KnowledgeEntry, GeneratedResponse (transitórios)
├── application/
│   ├── ports.py                  # SupportPlatformPort, LLMProviderPort, KnowledgePort
│   └── use_cases/
│       ├── receive_webhook_event.py     # dedup (cache) + dispara background
│       ├── process_incoming_message.py  # orquestra
│       ├── retrieve_context.py           # histórico (Digisac API) + KB (BM25)
│       ├── generate_response.py
│       └── dispatch_response.py          # send_reply / escalate
└── adapters/
    ├── inbound/webhooks/digisac_router.py
    ├── outbound/platform/digisac_client.py   # parse / send_reply / escalate / fetch_history
    ├── outbound/llm/openrouter_client.py
    ├── outbound/knowledge/excel_repository.py
    └── dedup/ttl_cache.py                     # cache in-memory (cachetools)

personas/support_persona.md
docs/                              # base de conhecimento (.xlsx)
tests/{unit,integration}/
docker-compose.yml · Dockerfile · pyproject.toml · .env.example
```

Removidos do desenho anterior: `adapters/persistence/`, `alembic/`, `scripts/purge_expired.py`
(sem banco / sem PII em repouso).

**Structure Decision**: web service único em Clean Architecture (3 camadas). Toda integração
externa fica em `adapters/`, atrás de portas em `application/ports.py`. O cache de dedup é um
adapter simples e substituível (troca por Redis/DB é trocar adapter, sem tocar em regra).

## Manutenção do diagrama de fluxo (pedido do usuário)

[`design/flow-sequence.md`](./design/flow-sequence.md) é **artefato vivo**: qualquer mudança no fluxo
escopado atualiza o mermaid na mesma alteração. Registrado em `CLAUDE.md §6`.

## Complexity Tracking

Sem complexidade adicional. Os dois desvios de constituição estão na tabela “Desvios
justificados” acima (removem componentes, não adicionam).
