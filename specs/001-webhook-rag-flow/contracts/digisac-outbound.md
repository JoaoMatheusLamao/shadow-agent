# Contrato — Chamadas de Saída (Serviço → API Digisac)

Auth: `Authorization: Bearer ${DIGISAC_API_TOKEN}` (Personal Access Token).
Base URL: `${DIGISAC_BASE_URL}` (host da conta). Todo I/O com **timeout + retry** (D8).
Campos/paths marcados **(confirmar em homologação)** — isolados no adapter.

## 1. Enviar resposta ao cliente

```
POST {DIGISAC_BASE_URL}/api/v1/messages
Authorization: Bearer <token>
Content-Type: application/json

{
  "text": "<resposta gerada e sanitizada>",
  "type": "chat",
  "contactId": "c-123",
  "serviceId": "s-456"
}
```

- `contactId`/`serviceId` vêm do webhook de entrada (mesma conversa/conexão).
- **Sanitização obrigatória** antes do envio (CLAUDE.md §3.3): validar que `text` é
  string não vazia, dentro de limite de tamanho; nunca enviar saída bruta da LLM sem
  checagem de formato.
- Sucesso → log `reply_sent` (correlationId, ticketId). Nada é persistido.

## 2. Escalonar (transferir ticket para humano)

```
POST {DIGISAC_BASE_URL}/api/v1/contacts/{contactId}/ticket/transfer
Authorization: Bearer <token>
Content-Type: application/json

{
  "departmentId": "${DIGISAC_ESCALATION_DEPARTMENT_ID}",
  "comments": "Escalonado automaticamente: sem contexto suficiente / baixa confiança."
}
```

- Usado quando não há conhecimento relevante ou `can_answer=false` (D7).
- Sucesso → `Conversation.status = escalated`.

## 3. Buscar histórico do ticket (fonte primária do contexto — sem banco)

```
GET {DIGISAC_BASE_URL}/api/v1/messages?where[contactId]={contactId}
Authorization: Bearer <token>
```

- **Fonte primária** do histórico da sessão (não há base local). Filtrar pelo ticket aberto
  (`data.ticketId`), ordenar cronologicamente, limitar a `MAX_CONTEXT_MESSAGES`.
- Roda no background (não no ACK), com timeout+retry.
- **Fallback**: se falhar após retries, usar **apenas a mensagem atual** como contexto
  (degrade graceful) + log `step_failed`; não travar. **(confirmar filtro por ticket)**

## Tratamento de erro (todas as chamadas)

| Situação | Ação |
|---|---|
| Timeout / 5xx | retry 2–3x com backoff exponencial curto |
| Falha após retries | **log `step_failed`** (`failed_at_step`, `error_type`, `correlationId`) — sem tabela de replay; Digisac pode reentregar |
| 4xx de autenticação | log de erro de infra (token inválido/expirado) |
