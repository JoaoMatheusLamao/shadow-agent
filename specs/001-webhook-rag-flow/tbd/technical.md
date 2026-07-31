# TBD — Definições Técnicas

> **Para quem:** time técnico / integração. **Objetivo:** fechar as incógnitas técnicas
> antes de gerar as tarefas (`/speckit-tasks`) e implementar. Complementa o `product.md`
> (produto). Base: `research.md`, `plan.md`, `contracts/`.
>
> 🔴 bloqueia implementar a integração · 🟡 confirmar durante a implementação/homologação ·
> 🟢 configuração/refino. Última atualização: 2026-07-30.

---

## 1. Integração Digisac — API e Webhook 🔴
A doc pública (Postman/GitBook) não expôs 100% dos campos. Precisamos validar na **conta
real** (idealmente um ambiente de homologação).

- **1.1 Acesso** 🔴 — `DIGISAC_BASE_URL` (host da conta, ex.: `https://<sub>.digisac.chat`)
  e um **Personal Access Token** gerado (Conta → API → Personal Access Tokens).
  - **Resposta:**
- **1.2 Configuração do webhook** 🔴 — cadastrar a URL `POST /webhooks/digisac/{secret}`,
  tipo "Geral", e **quais eventos** assinar (confirmar que `message.created` é o correto
  para "mensagem nova do cliente").
  - **Resposta:**
- **1.3 Formato do payload do webhook** 🔴 — confirmar nomes/valores reais: campo que
  distingue mensagem do cliente vs. do agente (`isFromMe`? `fromMe`?), `type` para texto,
  e os IDs (`data.id`, `contactId`, `serviceId`, `ticketId`).
  - **Resposta:**
- **1.4 Enviar resposta** 🔴 — confirmar `POST /api/v1/messages` e o body exato
  (`text`, `type`, `contactId`, `serviceId`?). O `serviceId` é obrigatório no envio?
  - **Resposta:**
- **1.5 Buscar histórico do ticket** 🔴 — confirmar `GET /api/v1/messages?where[contactId]=`
  e **como filtrar pelo ticket aberto** (`ticketId`), ordenação e paginação.
  - **Resposta:**
- **1.6 Transferir/escalar** 🟡 — confirmar `POST /api/v1/contacts/{contactId}/ticket/transfer`,
  body e o `DIGISAC_ESCALATION_DEPARTMENT_ID` (ID do departamento humano — depende da
  decisão de produto §5.2 do `product.md`).
  - **Resposta:**
- **1.7 Prevenção de loop** 🔴 — confirmar que a mensagem que **nós** enviamos via API gera
  webhook com `isFromMe=true` (para o bot não responder à própria resposta).
  - **Resposta:**
- **1.8 Segurança do webhook** 🟡 — confirmar que o Digisac **não** oferece assinatura/HMAC
  (usaremos secret na URL). Há allowlist de IP disponível?
  - **Resposta:**
- **1.9 Rate limits** 🟡 — limites de requisição da API do Digisac (envio/listagem)?
  - **Resposta:**
- **1.10 Ambiente de teste** 🟡 — existe sandbox/homologação para testar sem enviar mensagem
  a clientes reais? Um número/conexão de teste?
  - **Resposta:**

## 2. Modelo LLM (OpenRouter) 🟡

- **2.1 Slug do modelo** 🟡 — confirmar o ID exato do default em `openrouter.ai/models`
  (proposto: `google/gemini-3.1-flash-lite`).
  - **Resposta:**
- **2.2 Validar PT-BR** 🟡 — testar qualidade/tom em português com casos reais antes de
  fixar; comparar com fallback `openai/gpt-5.4-mini`.
  - **Resposta:**
- **2.3 Fallback** 🟢 — a troca de modelo é manual (env `OPENROUTER_MODEL`) ou queremos
  fallback automático em erro/baixa confiança? (MVP: manual.)
  - **Resposta:**
- **2.4 Chave e billing** 🟡 — `OPENROUTER_API_KEY`, limite de gasto/alertas de custo.
  - **Resposta:**

## 3. Recuperação de conhecimento (Excel/BM25) 🟡

- **3.1 `KNOWLEDGE_MIN_SCORE`** 🔴 — limiar de relevância que decide **responder vs.
  escalar**. Só dá para calibrar com o Excel real (depende de `product.md` §2.1).
  - **Resposta:**
- **3.2 `KNOWLEDGE_TOP_K`** 🟢 — nº de trechos passados à LLM (default 4). Confirmar.
  - **Resposta:**
- **3.3 Colunas do Excel** 🟡 — confirmar nomes reais das colunas (mapear para
  `pergunta`/`resposta`/`fonte`). Ver `contracts/knowledge-excel.md`.
  - **Resposta:**
- **3.4 Recarga** 🟢 — no MVP a base recarrega no deploy. Precisamos de recarga sem
  redeploy (hot-reload/endpoint admin)?
  - **Resposta:**

## 4. Parâmetros de execução 🟢
Defaults propostos — confirmar.

- **4.1 `MAX_CONTEXT_MESSAGES`** (histórico do ticket) = 30. **Resposta:**
- **4.2 `DEDUP_TTL_SECONDS`** (cache de dedup em memória) = 86400 (24h). **Resposta:**
- **4.3 `LLM_TIMEOUT_SECONDS`** = 20; nº de retries por etapa = 2–3. **Resposta:**

## 5. Sem persistência — trade-offs a ratificar 🟡
Decisão já tomada (sem banco); confirmar que os limites são aceitáveis para o negócio.

- **5.1 Dedup volátil** 🟡 — em reinício/deploy o cache zera; uma reentrega logo após
  restart pode gerar **resposta duplicada**. Aceitável no MVP?
  - **Resposta:**
- **5.2 Perda em crash** 🟡 — se o processo cair entre o ACK e o fim do processamento, a
  mensagem pode se perder (o Digisac pode reentregar). Aceitável?
  - **Resposta:**
- **5.3 Gatilho de reversão** 🟢 — a partir de que ponto reintroduzimos um store durável
  (Redis/DB) para dedup/replay?
  - **Resposta:**

## 6. Hospedagem, observabilidade e LGPD 🟡

- **6.1 Plataforma** 🟡 — confirmar **DigitalOcean App Platform (~US$5/mês)**; criar conta/
  projeto. Alternativa: Hetzner CX23. Região preferencial?
  - **Resposta:**
- **6.2 Segredos** 🟡 — onde guardar (secret manager da plataforma / variáveis de ambiente).
  Nada em código/`.env` commitado.
  - **Resposta:**
- **6.3 Logs — agregação e retenção** 🟡 — onde consultar logs por `correlationId`? Os logs
  guardam **IDs de contato/ticket** (dado pessoal, mesmo sem conteúdo) — definir **retenção
  de logs** para LGPD (ex.: 30/90 dias).
  - **Resposta:**
- **6.4 Métricas mínimas** 🟢 — expor latência (ACK e ponta a ponta), taxa de escalonamento,
  taxa de erro? Onde?
  - **Resposta:**
- **6.5 Disponibilidade** 🟡 — o design é **instância única** (estado em memória). A meta de
  ≥99.9% do SDD é otimista — ratificar meta realista para o MVP (ou aceitar o gap).
  - **Resposta:**

## 7. Contratos de saída e processo 🟢

- **7.1 Saída estruturada da LLM** 🟡 — confirmar o formato `{answer, can_answer}` e a
  **sanitização** antes de enviar ao Digisac (tamanho máximo, sem markdown que o canal não
  renderiza, etc.).
  - **Resposta:**
- **7.2 Constituição do projeto** 🟢 — a constituição formal (`.specify/memory/constitution.md`)
  é um template. Preencher via `/speckit-constitution` ou aceitar o `CLAUDE.md §3` como
  fonte de princípios?
  - **Resposta:**

---

### Resumo dos bloqueadores 🔴 (travam a integração)
1. Acesso e contrato do Digisac: token/URL (§1.1), eventos do webhook (§1.2), payload real
   (§1.3), envio (§1.4), histórico (§1.5), loop (§1.7).
2. `KNOWLEDGE_MIN_SCORE` (§3.1) — depende do Excel real (`product.md` §2.1).

> Depois que os 🔴 daqui e do `product.md` estiverem respondidos, atualizamos a spec/plan
> nos pontos afetados e seguimos para `/speckit-tasks`.
