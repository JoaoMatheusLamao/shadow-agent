# Quickstart — Validação Ponta a Ponta

**Feature**: `001-webhook-rag-flow` · Prova que webhook → histórico do ticket (API Digisac)
→ conhecimento (Excel) → LLM → resposta funciona, **sem** Digisac/OpenRouter reais (fakes
das portas). **Sem banco.** Detalhes em `contracts/` e `data-model.md`.

## Pré-requisitos

- Python 3.12+, `uv`. (Sem Docker/Postgres — não há banco.)
- `.env` a partir de `.env.example` (abaixo).
- `docs/knowledge_base.xlsx` de exemplo (formato em `contracts/knowledge-excel.md`).
- `personas/support_persona.md` de exemplo.

## Variáveis de ambiente (`.env.example`)

```
DIGISAC_BASE_URL=https://<subdominio>.digisac.chat
DIGISAC_API_TOKEN=            # Personal Access Token (secret)
DIGISAC_WEBHOOK_SECRET=       # token na URL do webhook
DIGISAC_ESCALATION_DEPARTMENT_ID=
OPENROUTER_API_KEY=
OPENROUTER_MODEL=google/gemini-3.1-flash-lite   # confirmar slug em openrouter.ai/models
KNOWLEDGE_DIR=docs/
KNOWLEDGE_TOP_K=4
KNOWLEDGE_MIN_SCORE=          # calibrar
MAX_CONTEXT_MESSAGES=30
LLM_TIMEOUT_SECONDS=20
DEDUP_TTL_SECONDS=86400       # cache de dedup em memória
TENANT_ID=default
```

## Setup

```bash
uv sync
uv run uvicorn app.main:app --reload
```

## Cenários de validação

### C1 — Resposta fundamentada (happy path) — US1/US2, SC-001/SC-002
1. Garanta uma linha no Excel sobre um tópico (ex.: "prazo de entrega").
2. Envie um webhook simulado (`message.created`, isFromMe=false) com `text` sobre o tópico:
   ```bash
   curl -X POST "http://localhost:8000/webhooks/digisac/$DIGISAC_WEBHOOK_SECRET" \
     -H 'Content-Type: application/json' \
     -d '{"event":"message.created","data":{"id":"m1","text":"Qual o prazo de entrega?","type":"chat","isFromMe":false,"contactId":"c1","serviceId":"s1","ticketId":"t1"}}'
   ```
3. **Esperado**: `202` imediato; o fake do Digisac (outbound) registra **uma** resposta
   coerente com o Excel; log `reply_sent` com o `correlationId`.

### C2 — Contexto do ticket (via fake da API Digisac) — US2/SC-002
1. Configure o fake de `fetch_history` para o ticket `t1` retornar `["Quero assinar o
   plano."]`.
2. Envie `m2` "E no caso do anual?" (mesmo `t1`).
3. **Esperado**: a chamada capturada ao fake da LLM inclui o histórico buscado (msg
   anterior + atual), não só a última mensagem.

### C3 — Deduplicação in-memory — US3/SC-003
1. Envie o **mesmo** payload (`data.id=m1`) duas vezes seguidas (mesmo processo).
2. **Esperado**: apenas **uma** resposta; segundo POST → `200` + log `duplicate_ignored`.
3. *(Limitação conhecida: reiniciar o processo entre os envios pode gerar duplicata — cache
   volátil, ver research.md D3.)*

### C4 — Escalonamento sem contexto — US1/FR-004/D7
1. Envie mensagem sobre tópico **ausente** do Excel.
2. **Esperado**: nenhuma resposta automática; fake do Digisac registra **transfer**; log
   `escalated` com `reason=no_context`.

### C5 — Falha de infra e log — SC-005
1. Configure o fake da LLM para falhar (timeout) em todas as tentativas.
2. **Esperado**: após os retries, log `step_failed` com `failed_at_step=generate_response`;
   nenhuma resposta enviada. *(Sem tabela de replay — diagnóstico via log.)*

### C6 — Rastreabilidade por correlationId — US4/SC-004
1. Processe uma mensagem e capture o `correlationId` do log `webhook_received`.
2. **Esperado**: filtrando os logs por esse `correlationId`, todas as etapas aparecem.

## Testes automatizados

```bash
uv run pytest        # unit com fakes das portas (sem rede, sem banco)
uv run ruff check .
uv run mypy .
```

Cobertura mínima: use cases da Application, **dedup in-memory**, retry/log de falha
(CLAUDE.md §3.6).

> **LGPD**: não há banco nem PII em repouso ⇒ **não há job de expurgo**. Garanta apenas que
> os logs não contêm conteúdo de mensagem.
