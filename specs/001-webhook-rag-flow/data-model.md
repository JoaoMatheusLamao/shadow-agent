# Data Model — `001-webhook-rag-flow`

**Data**: 2026-07-30 · **Revisão**: 2026-07-30 — **sem persistência (sem banco)**.
Fonte: `spec.md` (Key Entities) + `research.md` (D3, D4, D8, D9).

> **Sem banco de dados neste estágio.** Não há tabelas, ORM nem migrations. As entidades
> abaixo são **objetos de domínio transitórios**, existentes apenas em memória durante o
> processamento de uma mensagem. O único estado que sobrevive entre requisições é o
> **cache de deduplicação em memória** (volátil). Falhas e rastreamento vivem em **logs
> estruturados**. Todas as entidades carregam `tenantId` (single-tenant, valor fixo) para
> evolução futura.

## Entidades de domínio (transitórias, camada Domain, puras)

### `DomainMessage`
Mensagem recebida, montada a partir do payload do webhook (adapter).

| Campo | Origem | Uso |
|---|---|---|
| external_message_id | `data.id` | chave de dedup (cache) |
| text | `data.text` | consulta ao conhecimento + prompt |
| contact_id | `data.contactId` | envio da resposta / busca de histórico |
| service_id | `data.serviceId` | envio da resposta (conexão) |
| ticket_id | `data.ticketId` | escopo da sessão (histórico) |
| correlation_id | gerado na borda | rastreamento em logs |

### `Conversation` (transitória)
Representa a sessão = ticket aberto. Montada em memória; **não persistida**.

| Campo | Uso |
|---|---|
| external_conversation_id (= ticket_id) | identifica a sessão |
| contact_id / service_id | roteamento da resposta |
| status | `open` \| `escalated` — usado só no fluxo/logs (não gravado) |

### `TicketHistory` (transitória)
Lista de mensagens do ticket, **buscada ao vivo na API do Digisac** (D4), ordenada por
tempo, limitada a `MAX_CONTEXT_MESSAGES`. Cada item: `{direction, text, created_at}`.
Fallback: só a mensagem atual, se a listagem falhar após retries.

### `KnowledgeEntry` (transitória, do Excel)
Carregada de `docs/*.xlsx` em memória (BM25). Campos: `title/question`, `content/answer`,
`source_ref`. Ver `contracts/knowledge-excel.md`.

### `Persona` (config versionada)
De `personas/*.md`. Campos: `name`, `tone`, `scope`, `escalation_policy`, `system_prompt`.
Nunca hardcoded (CLAUDE.md §2).

### `GeneratedResponse` (transitória)
Saída validada da LLM: `{ answer: str, can_answer: bool }`.

## Estado que sobrevive entre requisições

### Cache de deduplicação (in-memory, volátil) — D3
- Estrutura: `TTLCache` (ex.: `cachetools`) keyed por `external_message_id`, TTL ~24h.
- Semântica: `add_if_absent(id) -> bool`. Se já presente → duplicado → ignora.
- **Volátil**: perdido em reinício/crash ⇒ duplicidade possível pós-restart (limitação
  aceita). Não é um banco; não há garantia durável de idempotência.

## Registro de falhas e rastreamento — **logs** (não tabela) — D8/D10

Sem `failed_messages`/`processed_events`. Em vez disso, eventos estruturados de log:

| Evento de log | Campos (sem PII de conteúdo) |
|---|---|
| `webhook_received` | correlation_id, external_message_id, ticket_id |
| `duplicate_ignored` | correlation_id, external_message_id |
| `context_retrieved` | correlation_id, history_len, kb_hits |
| `escalated` | correlation_id, reason (`no_context`\|`low_confidence`) |
| `reply_sent` | correlation_id, ticket_id |
| `step_failed` | correlation_id, failed_at_step, error_type, retry_count |

Rastreabilidade ponta a ponta (SC-004) = filtrar logs por `correlation_id`.

## LGPD (D9)

**Sem PII em repouso** — nada de conteúdo de conversa é persistido pelo serviço. Job de
expurgo de 90 dias **não é necessário**. Regra mantida: **logs não contêm conteúdo** de
mensagem, apenas IDs/metadados/status.

## Relacionamentos (lógicos, em memória)

```
DomainMessage ──(ticket_id)── Conversation ──(fetch)── TicketHistory
DomainMessage.text ──(BM25)── KnowledgeEntry[]
(Persona + TicketHistory + KnowledgeEntry[]) ── LLM ── GeneratedResponse
dedup cache: external_message_id → visto? (volátil)
```
