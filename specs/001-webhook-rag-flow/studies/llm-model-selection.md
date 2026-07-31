# Estudo — Seleção de Modelo LLM (OpenRouter)

**Feature**: `001-webhook-rag-flow` · **Data**: 2026-07-30
**Resumo executivo**: eleito **`google/gemini-3.1-flash-lite`** (default), com fallback de
qualidade **`openai/gpt-5.4-mini`** e opção ultra-econômica **`deepseek/deepseek-v4-flash`**.
Modelo é **configurável por env** (`OPENROUTER_MODEL`). Ver decisão em `../research.md` D6.
**Custo do OpenRouter é pay-as-you-go e NÃO entra na conta de infra.**

---

## 1. Perfil de uso (o que importa aqui)

Suporte ao cliente **em PT-BR**, respostas curtas **fundamentadas por RAG** (base Excel),
boa aderência a **persona/tom**, **baixa alucinação**, **saída estruturada** (JSON
`{answer, can_answer}`), **baixa latência** e **baixo custo**. Benchmarks de código são
**irrelevantes** para este caso.

## 2. Comparação (jul/2026)

| Modelo | Preço in/out (US$/1M) | Pontos fortes | Observações |
|---|---|---|---|
| **google/gemini-3.1-flash-lite** ✅ default | **~$0.25 / $1.50** | **Multilíngue top da classe (MMMLU 88.9%)**; otimizado p/ latência; contexto 1M | Melhor custo-benefício p/ PT-BR + latência |
| openai/gpt-5.4-mini — fallback qualidade | ~$0.75 / $4.50 | Melhores benchmarks de conhecimento no tier budget (HLE 41.5) | ~3–6x mais caro; usar se a qualidade exigir |
| deepseek/deepseek-v4-flash — ultra-econômico | ~$0.14 / $0.28 | ~4x mais barato; contexto enorme | PT-BR menos documentado; **validar em teste** antes |
| anthropic/claude-haiku-4.5 | ~$1 / $5 | Ótima aderência a instrução | Mais caro que o Flash-Lite |

## 3. Decisão e racional

- **Default: `google/gemini-3.1-flash-lite`** — melhor **qualidade multilíngue por dólar**
  (MMMLU 88.9% ⇒ PT-BR forte), **latência baixa** (importante p/ a meta ponta a ponta),
  contexto de sobra para histórico do ticket + trechos da base, e preço baixo.
- **Fallback de qualidade**: `openai/gpt-5.4-mini` se surgirem falhas de qualidade/tom.
- **Opção ultra-econômica**: `deepseek/deepseek-v4-flash` (validar PT-BR antes de adotar).
- **Configurável**: trocar de modelo é mudar `OPENROUTER_MODEL` — sem tocar em código.

## 4. Estimativa de custo por resposta (referência)

Com ~3K tokens de entrada (persona + histórico + top-K da base) e ~400 de saída:
- `gemini-3.1-flash-lite` ≈ **US$0,0014/resposta** (~US$1,40 por 1.000 respostas).
- *(Custo OpenRouter é pay-as-you-go, à parte da infra — ver `hosting-cost-study.md`.)*

## 5. Pontos a confirmar (ver `../tbd/technical.md` §2)

- **Slug exato** do modelo em `openrouter.ai/models` (a página é SPA; confirmar o ID).
- **Validação de PT-BR** com casos reais antes de fixar o default.
- Decidir se o fallback é **manual** (env) ou automático (MVP: manual).
- `OPENROUTER_API_KEY` + limite/alertas de gasto.

## 6. Fontes

- BenchLM — Best Budget LLMs 2026: <https://benchlm.ai/blog/posts/best-budget-llms-2026>
- BenchLM — Gemini 3.1 Flash-Lite vs GPT-5.4 mini: <https://benchlm.ai/compare/gemini-3-1-flash-lite-vs-gpt-5-4-mini>
- TokenCost — Cheapest LLMs 2026 (V4-Flash vs Haiku/Nano/Flash-Lite): <https://tokencost.app/blog/deepseek-v4-flash-vs-haiku-nano-flash-lite>
- OpenRouter — Lowest-cost inference guide: <https://openrouter.ai/blog/tutorials/how-to-get-the-lowest-cost-llm-inference-on-openrouter/>
- IntuitionLabs — Low-Cost LLM comparison: <https://intuitionlabs.ai/articles/low-cost-llm-comparison>
