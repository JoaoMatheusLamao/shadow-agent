# Estudo — Hospedagem e Custo Mensal (infra do nosso projeto)

**Feature**: `001-webhook-rag-flow` · **Data**: 2026-07-30
**Escopo do custo**: apenas a **infra do nosso serviço rodando**. **OpenRouter e Digisac
NÃO entram** nesta conta (são custos de terceiros à parte).
**Resumo executivo**: eleito **DigitalOcean App Platform (~US$5/mês)**; alternativa mais
barata **Hetzner CX23 (~US$4,3/mês)**. Ver decisão em `../research.md` D11.

---

## 1. O que precisamos hospedar

Um **único container stateless** (FastAPI/Uvicorn, Python 3.12). **Sem banco de dados**
(decisão de MVP): estado em memória, dedup em cache volátil, logs estruturados. Volume
baixo/moderado. Portanto o alvo é: **1 serviço web, sempre-ligado, barato**.

## 2. Restrição técnica que decide a escolha

O processamento pesado (buscar histórico no Digisac + chamar a LLM) roda **em background,
depois** de o webhook já ter respondido `202` (`BackgroundTasks` do FastAPI). Consequências:

- A **instância precisa continuar viva** após responder o HTTP para terminar o trabalho.
- **Serverless scale-to-zero** (Google Cloud Run, AWS Lambda) **não serve bem**: a instância
  pode ser recuperada logo após a resposta (matando a tarefa em background) e o **cold
  start** ameaça a meta de ACK < 2s. Só funcionaria com `min-instances=1` (sempre-ligado),
  o que **anula a economia** do scale-to-zero.

➡ **Conclusão**: escolher um **container pequeno sempre-ligado**, não serverless por request.

## 3. Comparação (valores aproximados, jul/2026 — verificar no provedor)

| Plataforma | Custo/mês (só infra) | Fit p/ nosso padrão | Observações |
|---|---|---|---|
| **DigitalOcean App Platform** ✅ eleito | **~US$5** (fixo) | ✅ sempre-ligado, gerenciado | Billing previsível (sem per-request), deploy de container simples, logs integrados, sem gerenciar SO |
| **Hetzner CX23 (VPS)** — alternativa +barata | **~€3,99 (~US$4,3)** | ✅ sempre-ligado | Mais barato; porém **você gerencia** SO/Docker/reverse-proxy/TLS (mais trabalho de ops). ~3–5x mais barato que hyperscaler |
| Fly.io (shared-cpu-1x) | ~US$2 base (real ~US$8–25) | ✅ sempre-ligado | Free tier acabou; custo real sobe com egress/restarts |
| Railway (Hobby) | US$5 + uso | ✅ sempre-ligado | Sem banco agora ⇒ mais barato que a estimativa inicial; billing por uso menos previsível |
| Render | US$7 (starter) / US$0 free | ⚠ free hiberna | Starter previsível; o free tem **cold start** (ruim p/ ACK < 2s) |
| AWS App Runner | ~US$4–8 compute + overhead | ⚠ mais caro/complexo | Hyperscaler; caro/complexo para carga pequena e constante |
| Google Cloud Run | pay-per-use (~US$0 ocioso) | ❌ p/ este padrão | Scale-to-zero conflita com background pós-ACK; exigiria `min-instances=1` + cold start no ACK |
| AWS Lambda | pay-per-use | ❌ p/ este padrão | Mesmo problema do Cloud Run; modelo de execução não combina com tarefa em background pós-resposta |

## 4. Decisão

- **Eleito: DigitalOcean App Platform (basic, ~US$5/mês).** Melhor equilíbrio entre **custo
  baixo + simplicidade + billing previsível + sempre-ligado** (atende ao background
  pós-ACK), sem overhead de administrar um VPS. Logs integrados ajudam a rastreabilidade
  por `correlationId`.
- **Alternativa mais barata: Hetzner CX23 (~US$4,3/mês)** se o time aceitar gerenciar o VPS
  (Docker + systemd/Caddy para TLS).
- **Evitar neste padrão**: Google Cloud Run / AWS Lambda (scale-to-zero vs. background
  pós-ACK) e AWS App Runner (custo/complexidade para carga pequena constante).

## 5. Estimativa final

- **Infra do MVP: ~US$5/mês** (DigitalOcean App Platform) — podendo cair a **~US$4,3/mês**
  (Hetzner). **Sem banco ⇒ um único componente pago.**
- Fora desta conta (terceiros, por decisão do usuário): **OpenRouter** (pay-as-you-go por
  token) e **Digisac** (assinatura da plataforma de atendimento).

## 6. Pontos a confirmar (ver `../tbd/technical.md` §6)

- Conta/projeto no provedor e **região** preferencial (relevante p/ latência e, se um dia
  houver persistência, para LGPD/residência de dados).
- **Retenção de logs** (os logs guardam IDs de contato/ticket = dado pessoal, mesmo sem
  conteúdo) — definir política (ex.: 30/90 dias).
- Onde guardar **segredos** (secret manager da plataforma).
- **Disponibilidade**: o design é de **instância única** (estado em memória) — ratificar
  meta realista (a de ≥99.9% do SDD é otimista para instância única).

## 7. Fontes

- ExpressTech — Render vs Railway vs Fly.io (2026): <https://expresstech.io/render-vs-railway-vs-fly-io-2026-pricing-showdown/>
- ExpressTech — Fly.io alternatives após o fim do free tier (2026): <https://expresstech.io/7-fly-io-alternatives-in-2026-real-pricing-after-the-free-tier-died/>
- Forasoft — AWS vs DigitalOcean vs Hetzner (2026): <https://www.forasoft.com/blog/article/aws-vs-digitalocean-vs-hetzner-1302>
- Sliplane — Google Cloud Run alternatives (2026): <https://sliplane.io/blog/5-awesome-google-cloud-run-alternatives>
- Sliplane — AWS App Runner alternatives (2026): <https://sliplane.io/blog/5-awesome-aws-app-runner-alternatives>
- Render — Railway vs DigitalOcean App Platform (2026): <https://render.com/articles/railway-vs-digitalocean-app-platform-pricing-reliability-production-risk>
