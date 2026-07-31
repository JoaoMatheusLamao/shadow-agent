# Contrato — Webhook de Entrada (Digisac → Serviço)

**Endpoint exposto pelo serviço**

```
POST /webhooks/digisac/{webhook_secret}
Content-Type: application/json
```

- `{webhook_secret}` deve casar com `DIGISAC_WEBHOOK_SECRET`. Se não casar → `200` vazio
  (rejeição silenciosa, sem processar).
- Responde **`202 Accepted`** ao aceitar um evento novo e válido; **`200`** para
  ignorados/duplicados. Nunca bloqueia esperando o processamento (ACK < 2s).

## Payload aceito (Digisac `message.created`)

Formato observado nas libs de cliente (campos exatos **a confirmar em homologação** —
isolados no parser do adapter):

```json
{
  "event": "message.created",
  "data": {
    "id": "e2a1...-msg-id",
    "text": "Qual o prazo de entrega do plano anual?",
    "type": "chat",
    "isFromMe": false,
    "contactId": "c-123",
    "serviceId": "s-456",
    "ticketId": "t-789",
    "timestamp": "2026-07-30T18:20:00Z"
  },
  "webhookId": "wh-abc",
  "timestamp": "2026-07-30T18:20:01Z"
}
```

## Regras de aceitação na borda (antes de qualquer processamento)

| Condição | Ação |
|---|---|
| `webhook_secret` inválido | `200` vazio, descarta |
| `event != "message.created"` | `200`, ignora |
| `data.isFromMe == true` (mensagem do agente/bot) | `200`, ignora |
| `data.text` vazio/ausente (mídia/áudio) | `200`, ignora (fora de escopo do MVP) |
| `data.id` já no **cache de dedup (in-memory, TTL)** | `200`, idempotente best-effort (não reprocessa) |
| válido e novo | marca `data.id` no cache, dispara BackgroundTask, `202` |

> **Sem banco**: não há `processed_events`/`messages` persistidos. A dedup é um **cache em
> memória com TTL** (volátil — duplicidade possível pós-restart, ver research.md D3). Nada do
> payload é gravado em repouso.

## Mapeamento para o domínio (feito no adapter, em memória)

| Campo Digisac | Domínio (`DomainMessage`) |
|---|---|
| `data.id` | `external_message_id` (chave de dedup) |
| `data.text` | `text` (mensagem atual) |
| `data.contactId` | `contact_id` (envio + busca de histórico) |
| `data.serviceId` | `service_id` (conexão p/ envio) |
| `data.ticketId` | `ticket_id` (escopo da sessão) |

> Tipos específicos do Digisac **não** vazam além do adapter (CLAUDE.md §3.4).
