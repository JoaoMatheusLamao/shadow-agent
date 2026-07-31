# TBD — Definições de Produto (visão de PO)

> **Para quem:** Product Owner. **Objetivo:** decidir o *comportamento e o conteúdo* do
> assistente de IA antes de implementar. São decisões de **produto** (o que o bot faz e
> como conversa), não técnicas.
>
> **Como usar:** responda cada item no campo **Resposta:**. Itens marcados 🔴 **bloqueiam**
> uma implementação de qualidade; 🟡 são importantes mas não travam o início; 🟢 são
> refinamentos.
>
> Contexto do produto: assistente de IA que responde clientes **pelo Digisac**, usando uma
> **base de conhecimento (Excel)** e respeitando uma **persona** definida. Projeto em MVP,
> single-tenant (uma empresa). Última atualização: 2026-07-30.

---

## 1. Persona do assistente 🔴
O "jeito" do bot. Hoje **não existe** — precisa ser definida (vai virar um arquivo de
configuração versionado).

- **1.1 Nome e identidade** — o bot tem nome? Como se apresenta?
  - **Resposta:**
- **1.2 Tom de voz** — formal, informal, amigável, direto? Usa emojis? Usa "você"/"senhor(a)"?
  - **Resposta:**
- **1.3 Transparência (é uma IA?)** 🔴 — o bot deve deixar claro que é um assistente
  virtual/IA, ou se passar por atendente humano? *(Tem implicação ética/legal — recomenda-se
  transparência.)*
  - **Resposta:**
- **1.4 Escopo de atuação** — sobre o que ele PODE falar (ex.: dúvidas de produto, prazos,
  faturamento) e o que ele NÃO pode (ex.: negociar desconto, dar orientação jurídica/médica/
  financeira, prometer prazos fora da base)?
  - **Resposta:**
- **1.5 Limites/proibições** — há assuntos que ele nunca deve responder e sempre escalar
  (reclamação grave, cancelamento, dados sensíveis, cobrança)?
  - **Resposta:**
- **1.6 Saudação e encerramento** — deve cumprimentar no início? Se despedir? Perguntar se
  ajudou?
  - **Resposta:**

## 2. Base de conhecimento (Excel) 🔴
O bot só responde com base neste conteúdo.

- **2.1 Quem fornece o Excel e quando?** 🔴 — precisamos do arquivo real para começar.
  - **Resposta:**
- **2.2 Que tópicos ele cobre?** — lista de assuntos que a base deve responder no MVP.
  - **Resposta:**
- **2.3 Quem mantém/atualiza e com que frequência?** — o conteúdo muda? Quem edita?
  - **Resposta:**
- **2.4 Estrutura do conteúdo** — funciona como perguntas→respostas (FAQ) ou textos por
  tema? *(Isso ajuda a organizar as colunas da planilha.)*
  - **Resposta:**

## 3. Comportamento do atendimento 🔴
Como o bot se encaixa no fluxo real de suporte.

- **3.1 Bot × humano no mesmo atendimento** 🔴 — se um **atendente humano já assumiu** o
  ticket, o bot deve **parar de responder** automaticamente? Como saber que um humano
  assumiu?
  - **Resposta:**
- **3.2 Horário** — o bot responde 24/7 ou só fora do horário comercial (quando não há
  humano)? Há mensagem diferente fora do horário?
  - **Resposta:**
- **3.3 Cliente pede humano** — se o cliente digitar "quero falar com atendente", o bot
  escala na hora?
  - **Resposta:**
- **3.4 Quantas tentativas antes de desistir?** — se o bot não resolve na 1ª/2ª interação,
  ele insiste ou passa para humano?
  - **Resposta:**

## 4. Mensagens que não são texto 🔴
No Digisac (WhatsApp/etc.) o cliente manda **áudio, imagem, PDF, figurinha, localização**.
Hoje o desenho **ignora** essas mensagens (o bot não responde nada).

- **4.1 O que fazer com mídia/áudio?** 🔴 — ignorar (arriscado: cliente fica sem resposta),
  responder pedindo texto ("não consigo ouvir áudios, pode escrever?"), ou escalar para
  humano?
  - **Resposta:**

## 5. Quando escalar para humano 🟡
Hoje o bot escala quando **não encontra resposta na base** ou **não tem confiança**.

- **5.1 Avisar o cliente ao escalar?** 🟡 — quando transfere para humano, o bot manda uma
  mensagem tipo "vou te encaminhar para um atendente"? Qual texto?
  - **Resposta:**
- **5.2 Para qual time/departamento** o atendimento vai quando escala? (Existe uma fila/
  departamento no Digisac para isso?)
  - **Resposta:**
- **5.3 Outros gatilhos de escalonamento** — além de "não sei responder", deve escalar em
  casos como: reclamação, palavrão/cliente irritado, pedido de cancelamento, assunto
  financeiro?
  - **Resposta:**

## 6. Canais e escopo do MVP 🟡

- **6.1 Quais canais do Digisac** entram no MVP? (WhatsApp, Instagram, etc. — o Digisac
  agrega vários.)
  - **Resposta:**
- **6.2 Uma empresa só?** — confirmamos single-tenant (um cliente/empresa) neste estágio?
  - **Resposta:**
- **6.3 Idioma** — só PT-BR? O que fazer se o cliente escrever em outra língua?
  - **Resposta:**

## 7. Metas de sucesso do negócio 🟢
Ajuda a medir se o produto está funcionando (não bloqueia implementar).

- **7.1 Meta de resolução automática** — que % dos atendimentos esperamos que o bot resolva
  sozinho (sem humano)?
  - **Resposta:**
- **7.2 Taxa de escalonamento aceitável** — a partir de quanto "escala demais" é problema?
  - **Resposta:**
- **7.3 O que medir** — satisfação do cliente? tempo de resposta? algo mais?
  - **Resposta:**

---

### Resumo dos bloqueadores 🔴 (o que trava começar com qualidade)
1. Persona — tom, transparência (é IA?), escopo/limites (§1).
2. Excel de conhecimento real + tópicos cobertos (§2.1, §2.2).
3. Regra bot × humano no mesmo ticket (§3.1) + pedido de humano (§3.3).
4. O que fazer com mensagens de mídia/áudio (§4.1).

> As respostas destes itens viram: o arquivo de **persona**, o **Excel** em `docs/`, e
> regras de comportamento que entram na spec antes de gerar as tarefas de implementação.
