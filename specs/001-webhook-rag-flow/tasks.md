---
description: "Task list — Fluxo Ponta a Ponta (Digisac → Conhecimento → LLM → Resposta)"
---

# Tasks: Fluxo Ponta a Ponta — Digisac → Conhecimento → LLM → Resposta

**Input**: Design documents from `specs/001-webhook-rag-flow/`
**Prerequisites**: `plan.md`, `spec.md`, `research.md`, `data-model.md`, `contracts/`
**Stack**: Python 3.12 · FastAPI/Uvicorn · httpx · openpyxl + rank_bm25 · pydantic/pydantic-settings ·
tenacity · cachetools · structlog · BackgroundTasks. **Sem banco** (dedup em memória, logs).
**Testes**: obrigatórios (CLAUDE.md §3.6) — incluídos por história.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: pode rodar em paralelo (arquivos diferentes, sem dependência pendente)
- **[Story]**: US1/US2/US3/US4 (Setup/Foundational/Polish não têm label)

> ⚠️ **Dependências de TBD** (ver `tbd/product.md` e `tbd/technical.md`): o **conteúdo** da
> persona (`personas/support_persona.md`) e do **Excel** (`docs/*.xlsx`), o valor de
> `KNOWLEDGE_MIN_SCORE`, e os **campos/endpoints reais do Digisac** dependem de respostas
> pendentes. As tarefas abaixo implementam o **comportamento especificado** e usam
> placeholders/valores documentados; os pontos a confirmar estão marcados `⟨TBD⟩`.

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Inicialização do projeto e estrutura.

- [ ] T001 Criar a estrutura de pastas do projeto (`app/`, `app/domain/`, `app/application/use_cases/`, `app/adapters/{inbound/webhooks,outbound/platform,outbound/llm,outbound/knowledge,dedup}/`, `app/common/`, `personas/`, `docs/`, `tests/{unit,integration}/`) com `__init__.py` conforme `plan.md`
- [ ] T002 Inicializar projeto Python com `uv` e `pyproject.toml` (deps: fastapi, uvicorn, httpx, openpyxl, rank-bm25, pydantic, pydantic-settings, tenacity, cachetools, structlog; dev: pytest, pytest-asyncio, ruff, mypy)
- [ ] T003 [P] Configurar `ruff` (lint+format) e `mypy` (modo estrito) em `pyproject.toml`
- [ ] T004 [P] Criar `.env.example` com todas as variáveis de `quickstart.md` (DIGISAC_*, OPENROUTER_*, KNOWLEDGE_*, MAX_CONTEXT_MESSAGES, DEDUP_TTL_SECONDS, LLM_TIMEOUT_SECONDS, TENANT_ID)
- [ ] T005 [P] Criar `Dockerfile` (container único stateless, Uvicorn) para deploy no DigitalOcean App Platform
- [ ] T006 [P] Criar `README` de execução local (uv sync, uvicorn, pytest) em `specs/001-webhook-rag-flow/quickstart.md` já existe — apenas adicionar `README.md` na raiz apontando para ele

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Núcleo do qual todas as histórias dependem. **⚠️ Nenhuma história começa antes disto.**

- [ ] T007 [P] Implementar `app/config.py` com `Settings` (pydantic-settings) carregando todas as env vars, com defaults documentados (DEDUP_TTL_SECONDS=86400, MAX_CONTEXT_MESSAGES=30, LLM_TIMEOUT_SECONDS=20, KNOWLEDGE_TOP_K=4)
- [ ] T008 [P] Definir entidades de domínio puras em `app/domain/entities.py` (DomainMessage, Conversation, HistoryItem, TicketHistory, KnowledgeEntry, Persona, GeneratedResponse, DeliveryResult) — sem SDK/HTTP/ORM (ver `data-model.md`)
- [ ] T009 [P] Definir as portas (Protocols) em `app/application/ports.py` (SupportPlatformPort, KnowledgePort, LLMProviderPort) conforme `contracts/ports.md`
- [ ] T010 [P] Configurar logging estruturado (structlog/JSON) e binding de `correlationId` em `app/common/logging.py` — logs sem conteúdo de mensagem (CLAUDE.md §3.3/§3.5)
- [ ] T011 [P] Implementar helper de retry com backoff exponencial curto (tenacity) e timeout em `app/common/retry.py` (2–3 tentativas)
- [ ] T012 Criar o app FastAPI e a fiação de dependências (DI) em `app/main.py` (instancia adapters e injeta nas portas; registra o router; sem lógica de negócio)

**Checkpoint**: fundação pronta — histórias podem começar.

---

## Phase 3: User Story 1 - Responder com base em conhecimento (Priority: P1) 🎯 MVP

**Goal**: Uma mensagem do cliente chega pelo webhook → recupera conhecimento do Excel → gera
resposta com persona (LLM) → envia pelo Digisac; se não há conhecimento relevante ou baixa
confiança, escala. (Contexto = mensagem atual; histórico do ticket é a US2.)

**Independent Test**: POST de um `message.created` cujo tópico existe no Excel → o fake do
Digisac registra **uma** resposta coerente; tópico ausente → registra **transfer** (escala).

### Tests for User Story 1 (obrigatórios) ⚠️

- [ ] T013 [P] [US1] Teste unitário do `excel_repository` (carga do `.xlsx` + BM25 top-K + limiar → lista vazia quando nada relevante) em `tests/unit/test_excel_repository.py`
- [ ] T014 [P] [US1] Teste unitário de `generate_response` com **fake LLM** (parse/validação do JSON `{answer, can_answer}`; erro de formato após retries = falha) em `tests/unit/test_generate_response.py`
- [ ] T015 [P] [US1] Teste unitário de `dispatch_response` com **fake platform** (envia resposta quando `can_answer=true`; escala quando `can_answer=false` ou KB vazia) em `tests/unit/test_dispatch_response.py`
- [ ] T016 [P] [US1] Teste de integração webhook→resposta com fakes (happy path + escalonamento sem contexto) em `tests/integration/test_us1_reply_flow.py`

### Implementation for User Story 1

- [ ] T017 [P] [US1] Implementar `KnowledgePort` via Excel+BM25 em `app/adapters/outbound/knowledge/excel_repository.py` (carga no startup de `docs/*.xlsx`, mapeamento de colunas conforme `contracts/knowledge-excel.md`, `retrieve()` top-K com `KNOWLEDGE_MIN_SCORE`)
- [ ] T018 [P] [US1] Implementar loader de persona em `app/adapters/outbound/persona/persona_loader.py` (lê `personas/*.md` → `Persona`)
- [ ] T019 [P] [US1] Implementar `LLMProviderPort` (OpenRouter) em `app/adapters/outbound/llm/openrouter_client.py` (httpx async, `OPENROUTER_MODEL`, timeout, prompt = persona+contexto+trechos, saída JSON validada com pydantic) — usa `app/common/retry.py`
- [ ] T020 [US1] Implementar `SupportPlatformPort` (Digisac) — `parse_incoming_event` + `send_reply` (`POST /api/v1/messages`) + `escalate` (`ticket/transfer`) em `app/adapters/outbound/platform/digisac_client.py` conforme `contracts/{webhook-inbound,digisac-outbound}.md` ⟨TBD: campos/endpoint reais⟩ (sem `fetch_history` ainda)
- [ ] T021 [US1] Implementar `retrieve_context` (só KB nesta história: consulta `KnowledgePort` com o texto atual) em `app/application/use_cases/retrieve_context.py`
- [ ] T022 [US1] Implementar `generate_response` em `app/application/use_cases/generate_response.py` (monta prompt persona+contexto+trechos, chama `LLMProviderPort`, valida/sanitiza saída)
- [ ] T023 [US1] Implementar `dispatch_response` em `app/application/use_cases/dispatch_response.py` (envia via `send_reply` **com sanitização** ou `escalate`; regra D7: KB vazia OU `can_answer=false` → escala)
- [ ] T024 [US1] Implementar `process_incoming_message` (orquestra retrieve→generate→dispatch, com retry por etapa e log `step_failed` em falha final) em `app/application/use_cases/process_incoming_message.py`
- [ ] T025 [US1] Implementar o endpoint de webhook em `app/adapters/inbound/webhooks/digisac_router.py`: valida secret na URL, filtra `event=message.created`/`isFromMe=false`/texto não vazio (edge cases da spec → `200` ignora), gera `correlationId`, ACK `202` e dispara `process_incoming_message` via `BackgroundTasks`
- [ ] T026 [US1] Criar `personas/support_persona.md` (persona inicial: tom, escopo, limites, política de escalonamento) ⟨TBD: conteúdo final do PO — `tbd/product.md` §1⟩
- [ ] T027 [US1] Adicionar `docs/knowledge_base.xlsx` de exemplo no formato de `contracts/knowledge-excel.md` ⟨TBD: base real — `tbd/product.md` §2⟩

**Checkpoint**: US1 funcional e testável de forma independente (MVP).

---

## Phase 4: User Story 2 - Responder considerando o histórico do ticket (Priority: P1)

**Goal**: Antes de gerar a resposta, reunir **todas as mensagens do ticket aberto** (buscadas na
API do Digisac) e passá-las como contexto à LLM.

**Independent Test**: ticket com mensagem anterior + nova pergunta de acompanhamento → a chamada
capturada à LLM inclui o histórico do ticket, não só a última mensagem.

### Tests for User Story 2 (obrigatórios) ⚠️

- [ ] T028 [P] [US2] Teste unitário de `fetch_history` (fake da API Digisac: monta `TicketHistory` filtrado pelo ticket, ordenado, limitado a `MAX_CONTEXT_MESSAGES`; fallback = só mensagem atual em falha) em `tests/unit/test_fetch_history.py`
- [ ] T029 [P] [US2] Teste de integração de acompanhamento (histórico influencia a resposta) em `tests/integration/test_us2_ticket_context.py`

### Implementation for User Story 2

- [ ] T030 [US2] Adicionar `fetch_history` ao `app/adapters/outbound/platform/digisac_client.py` (`GET /api/v1/messages?where[contactId]=` filtrado pelo ticket) com retry/timeout ⟨TBD: filtro por ticket⟩
- [ ] T031 [US2] Estender `retrieve_context` (`app/application/use_cases/retrieve_context.py`) para chamar `fetch_history` e compor o contexto (histórico + trechos KB); fallback graceful para só a mensagem atual em falha
- [ ] T032 [US2] Ajustar o prompt em `generate_response` (`app/application/use_cases/generate_response.py`) para incluir o histórico do ticket na ordem cronológica

**Checkpoint**: US1 + US2 funcionam de forma independente.

---

## Phase 5: User Story 3 - Deduplicação (não duplicar resposta) (Priority: P2)

**Goal**: Reentregas do mesmo evento (mesmo `data.id`) não geram resposta duplicada enquanto o
processo está vivo (dedup best-effort em memória, TTL).

**Independent Test**: enviar o mesmo `data.id` duas vezes no mesmo processo → apenas uma resposta;
2º POST retorna `200` + log `duplicate_ignored`.

### Tests for User Story 3 (obrigatórios) ⚠️

- [ ] T033 [P] [US3] Teste unitário do cache de dedup (`add_if_absent` retorna false na 2ª vez; expira após TTL) em `tests/unit/test_dedup_cache.py`
- [ ] T034 [P] [US3] Teste de integração de reenvio duplicado (uma única resposta entregue) em `tests/integration/test_us3_dedup.py`

### Implementation for User Story 3

- [ ] T035 [P] [US3] Implementar cache de dedup em memória (TTLCache/cachetools) com `add_if_absent(id)` em `app/adapters/dedup/ttl_cache.py` (TTL = `DEDUP_TTL_SECONDS`)
- [ ] T036 [US3] Implementar `receive_webhook_event` em `app/application/use_cases/receive_webhook_event.py` (checa/marca `data.id` no cache; se já visto → `duplicate_ignored`, não dispara background)
- [ ] T037 [US3] Integrar a dedup no `app/adapters/inbound/webhooks/digisac_router.py` (chamar `receive_webhook_event` antes de disparar o `BackgroundTasks`)

**Checkpoint**: US1 + US2 + US3 funcionam.

---

## Phase 6: User Story 4 - Rastreabilidade por correlationId (Priority: P3)

**Goal**: Cada mensagem é rastreável ponta a ponta pelos logs via `correlationId`; todas as etapas
emitem eventos estruturados nomeados.

**Independent Test**: processar uma mensagem e, filtrando os logs pelo `correlationId`, ver todas as
etapas (recepção → contexto → geração → entrega/escalonamento/falha).

### Tests for User Story 4 (obrigatórios) ⚠️

- [ ] T038 [P] [US4] Teste que captura logs e valida a presença dos eventos por `correlationId` (webhook_received, context_retrieved, reply_sent/escalated, step_failed) em `tests/integration/test_us4_traceability.py`

### Implementation for User Story 4

- [ ] T039 [US4] Propagar `correlationId` por todas as etapas (assinatura dos use cases / contextvar) e emitir os eventos estruturados nomeados de `data-model.md` (webhook_received, duplicate_ignored, context_retrieved, escalated, reply_sent, step_failed) nos pontos correspondentes de `app/application/use_cases/*` e `app/adapters/**`

**Checkpoint**: todas as histórias funcionam de forma independente.

---

## Phase 7: Polish & Cross-Cutting Concerns

- [ ] T040 [P] Adicionar `GET /health` (readiness/liveness) em `app/adapters/inbound/webhooks/` ou `app/main.py` para a plataforma de hosting
- [ ] T041 [P] Rodar e ajustar `ruff check .`, `ruff format .` e `mypy .` até zero erros
- [ ] T042 [P] Testes unitários adicionais de sanitização da saída da LLM (tamanho/format) em `tests/unit/test_sanitization.py`
- [ ] T043 Validar os cenários de `quickstart.md` (C1–C6) ponta a ponta com os fakes
- [ ] T044 Revisar segredos (nenhum em código/log), timeouts explícitos em todo I/O, e atualizar `design/flow-sequence.md` se algum comportamento mudou

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: sem dependências.
- **Foundational (Phase 2)**: depende do Setup — **bloqueia todas as histórias**.
- **User Stories (Phase 3–6)**: dependem da Foundational.
  - US1 é o MVP. US2 **estende** arquivos da US1 (`retrieve_context`, `generate_response`,
    `digisac_client`) → fazer **após** US1. US3 e US4 são majoritariamente independentes e podem
    ser paralelas à US2 (tocam arquivos diferentes: `dedup/`, `receive_webhook_event`, logging).
- **Polish (Phase 7)**: após as histórias desejadas.

### User Story Dependencies

- **US1 (P1)**: após Foundational. Sem dependência de outras histórias.
- **US2 (P1)**: após US1 (reusa/estende `retrieve_context`, `generate_response`, `digisac_client`).
- **US3 (P2)**: após Foundational; integra no router (T037 depende de T025 existir).
- **US4 (P3)**: após Foundational; consolida logs emitidos pelas demais (melhor após US1).

### Within Each User Story

- Testes primeiro (devem falhar antes da implementação) → adapters/portas → use cases → endpoint.

### Parallel Opportunities

- Setup: T003, T004, T005, T006 em paralelo.
- Foundational: T007–T011 em paralelo (T012 depende deles).
- US1: testes T013–T016 em paralelo; adapters T017/T018/T019 em paralelo (T020 depois; use cases dependem dos adapters).
- US3 pode correr em paralelo à US2 (arquivos distintos).

---

## Parallel Example: User Story 1

```bash
# Testes da US1 juntos (falham primeiro):
Task: "T013 Teste do excel_repository em tests/unit/test_excel_repository.py"
Task: "T014 Teste de generate_response em tests/unit/test_generate_response.py"
Task: "T015 Teste de dispatch_response em tests/unit/test_dispatch_response.py"
Task: "T016 Teste de integração webhook→resposta em tests/integration/test_us1_reply_flow.py"

# Adapters da US1 em paralelo:
Task: "T017 excel_repository (KnowledgePort/BM25)"
Task: "T018 persona_loader"
Task: "T019 openrouter_client (LLMProviderPort)"
```

---

## Implementation Strategy

### MVP First (só US1)

1. Phase 1 (Setup) → 2. Phase 2 (Foundational) → 3. Phase 3 (US1) → **PARAR e VALIDAR** US1
   isolada (webhook→KB→LLM→resposta / escalonamento). Demonstrar.

### Incremental

1. Setup + Foundational → base pronta.
2. US1 → testar → demo (MVP).
3. US2 (histórico do ticket) → testar → demo.
4. US3 (dedup) e US4 (rastreabilidade) → testar → demo.

### Bloqueadores de conteúdo (não de código)

- T026 (persona) e T027 (Excel) usam placeholders até as respostas de `tbd/product.md`.
- `KNOWLEDGE_MIN_SCORE` (T017/T023) e os campos do Digisac (T020/T030) confirmam-se com
  `tbd/technical.md` (homologação). O código roda com valores documentados; ajustar depois.

---

## Notes

- [P] = arquivos diferentes, sem dependência pendente.
- Testes obrigatórios (CLAUDE.md §3.6): use cases, dedup, retry/log de falha — cobertos por
  T013–T016, T028–T029, T033–T034, T038.
- Sem banco: dedup volátil (US3), histórico via API (US2), falhas em log (US4).
- Commit após cada tarefa ou grupo lógico; parar em qualquer checkpoint para validar a história.
