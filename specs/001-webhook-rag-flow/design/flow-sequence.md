# Diagrama de Sequência — Fluxo Escopado (MVP)

> **Artefato vivo.** Visão canônica do fluxo ponta a ponta; **mantenha atualizado** a cada
> mudança de comportamento (spec/plan/implementação), na mesma alteração. Referenciado em
> `CLAUDE.md` §6.
>
> **Feature**: `001-webhook-rag-flow` · **Última atualização**: 2026-07-30

Plataforma: **Digisac**. Conhecimento: **Excel em `docs/`** (BM25, sem pgvector). Sessão =
**ticket aberto**. Processamento assíncrono **in-process** (`BackgroundTasks`, sem fila).
**Sem banco/persistência**: dedup em **cache em memória (volátil)**; histórico do ticket
**buscado ao vivo na API do Digisac**; falhas em **logs estruturados**.

## Fluxo principal (dedup in-memory, escalonamento, falha)

```mermaid
sequenceDiagram
    autonumber
    actor Cliente
    participant DS as Digisac
    participant WH as Webhook Adapter<br/>(FastAPI, inbound)
    participant DC as Dedup Cache<br/>(in-memory, TTL)
    participant BG as ProcessIncomingMessage<br/>(BackgroundTask)
    participant HX as Digisac API<br/>(histórico do ticket)
    participant KB as Knowledge (Excel/BM25)
    participant LLM as OpenRouter<br/>(gemini-3.1-flash-lite)
    participant TX as Digisac Reply Adapter<br/>(outbound)
    participant LOG as Logs estruturados

    Cliente->>DS: Envia mensagem
    DS->>WH: POST /webhooks/digisac/{secret}<br/>{event, data:{id,text,contactId,serviceId,ticketId,isFromMe}}

    Note over WH: Validação de borda (síncrona, < 2s)
    alt secret inválido OU event≠message.created OU isFromMe=true OU text vazio
        WH-->>DS: 200 (ignora silenciosamente)
    else mensagem de contato válida
        WH->>DC: add_if_absent(data.id)?
        alt já visto (duplicado)
            DC-->>WH: false
            WH->>LOG: duplicate_ignored
            WH-->>DS: 200 (idempotente best-effort)
        else novo
            DC-->>WH: true (marcado, TTL ~24h)
            WH->>LOG: webhook_received (correlationId)
            WH->>BG: add_task(processar, correlationId)
            WH-->>DS: 202 Accepted (ACK imediato)

            Note over BG,TX: Processamento assíncrono (mesmo processo)
            BG->>HX: GET /api/v1/messages?where[contactId] (ticket aberto) [timeout+retry]
            alt histórico ok
                HX-->>BG: mensagens do ticket
            else falha após retries
                HX-->>BG: erro → usa só a mensagem atual (degrade) + LOG step_failed
            end
            BG->>KB: retrieve(texto) → top-K (BM25)

            alt nenhum trecho relevante (score < limiar)
                BG->>TX: escalar (motivo=no_context)
                TX->>DS: POST ticket/transfer → humano
                BG->>LOG: escalated
            else há conhecimento relevante
                BG->>LLM: generate(persona + histórico + trechos) [timeout+retry]
                alt LLM ok e can_answer=true
                    LLM-->>BG: {answer, can_answer:true}
                    BG->>TX: sendReply(answer sanitizada) [timeout+retry]
                    TX->>DS: POST /api/v1/messages {text, contactId, serviceId}
                    DS->>Cliente: Exibe resposta
                    BG->>LOG: reply_sent
                else can_answer=false
                    BG->>TX: escalar (motivo=low_confidence)
                    TX->>DS: POST ticket/transfer
                    BG->>LOG: escalated
                end
            end

            opt falha de infra em qualquer etapa (após retries)
                BG->>LOG: step_failed (failedAtStep, error_type, retryCount)
                Note over BG,LOG: Sem tabela de replay — diagnóstico via log;<br/>Digisac pode reentregar a mensagem
            end
        end
    end
```

## Legenda de decisões (ver `../research.md`)

- **secret na URL**: Digisac não tem HMAC — D2.
- **dedup in-memory TTL (volátil)**: sem banco; best-effort, duplicidade possível pós-restart — D3.
- **histórico via API do Digisac**: sem tabela local; fonte de verdade é o Digisac — D4.
- **BM25 sobre Excel**: recuperação lexical, sem embeddings — D5.
- **modelo**: `google/gemini-3.1-flash-lite` (configurável) — D6.
- **escalonamento**: sem contexto relevante ou `can_answer=false` → transfer — D7.
- **falhas → logs estruturados** (sem `failed_messages`): D8/D10.
- **sem PII em repouso**: LGPD simplificada — D9.
