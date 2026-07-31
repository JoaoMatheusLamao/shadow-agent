# Contrato — Portas (Interfaces da camada Application)

Interfaces que a Application define e os Adapters implementam. Expressas como
`typing.Protocol` (Python). Trocar Digisac/OpenRouter/Excel = trocar adapter, sem tocar
em Application/Domain (CLAUDE.md §3.4).

```python
from typing import Protocol
from app.domain.entities import (
    DomainMessage, RetrievedContext, GeneratedResponse,
    Persona, KnowledgeEntry, DeliveryResult,
)

class SupportPlatformPort(Protocol):
    """Digisac no MVP. Entrada (parse), histórico e saída (reply/escalate)."""
    def parse_incoming_event(self, raw_payload: dict) -> DomainMessage | None:
        """Converte o payload do webhook em mensagem de domínio.
        Retorna None se o evento deve ser ignorado (não é msg de contato)."""
        ...

    async def fetch_history(self, message: DomainMessage) -> list["HistoryItem"]:
        """Busca o histórico do ticket aberto na API do Digisac (fonte primária,
        pois não há base local). Fallback: [] → usa só a mensagem atual."""
        ...

    async def send_reply(self, message, text: str) -> DeliveryResult: ...

    async def escalate(self, message, reason: str) -> DeliveryResult:
        """Transfere o ticket para um departamento humano."""
        ...

class KnowledgePort(Protocol):
    """Base de conhecimento. No MVP: Excel + BM25 em memória."""
    def retrieve(self, query: str, tenant_id: str, top_k: int = 4) -> list[KnowledgeEntry]:
        """Retorna trechos relevantes acima do limiar; lista vazia se nada relevante."""
        ...

class LLMProviderPort(Protocol):
    """OpenRouter no MVP."""
    async def generate(
        self, context: RetrievedContext, persona: Persona
    ) -> GeneratedResponse:
        """Gera {answer, can_answer}; valida/sanitiza a saída antes de retornar."""
        ...
```

## Contratos de comportamento (testáveis com fakes)

- `parse_incoming_event`: dado um payload de `message.created` com `isFromMe=false`,
  retorna `DomainMessage`; para qualquer outro evento, retorna `None`.
- `KnowledgePort.retrieve`: entrada sem correspondência acima do limiar →
  `[]` (dispara escalonamento em `ProcessIncomingMessage`).
- `LLMProviderPort.generate`: em timeout/erro → levanta exceção de infra (dispara retry);
  saída fora do formato esperado após retries → falha de geração.
- `SupportPlatformPort.send_reply`/`escalate`: falha após retries → **log
  `step_failed`** (sem tabela de replay; Digisac pode reentregar).

## Use cases da Application (orquestração)

| Use case | Portas usadas | Resultado |
|---|---|---|
| `ReceiveWebhookEvent` | (cache dedup in-memory) | dedup best-effort + dispara background |
| `ProcessIncomingMessage` | todas | orquestra as etapas abaixo |
| `RetrieveContext` | SupportPlatformPort.fetch_history + KnowledgePort | histórico (API Digisac) + trechos (BM25) |
| `GenerateResponse` | LLMProviderPort | `GeneratedResponse` |
| `DispatchResponse` | SupportPlatformPort | envia resposta ou escala |

> Sem persistência: `ReceiveWebhookEvent` não grava nada — só consulta/atualiza o cache de
> dedup (volátil). Falhas de qualquer etapa são registradas em **log estruturado**.
