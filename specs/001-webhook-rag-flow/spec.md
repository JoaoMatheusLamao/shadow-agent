# Feature Specification: Fluxo Ponta a Ponta — Mensagem do Digisac, Conhecimento e Resposta

**Feature Branch**: `001-webhook-rag-flow`

**Created**: 2026-07-19

**Last Revised**: 2026-07-30

**Status**: Draft

**Input**: User description: "Fluxo macro do MVP do middleware de suporte com IA, agora com a plataforma de suporte definida como **Digisac**: quando um cliente interage no Digisac, o Digisac aciona um webhook para o nosso serviço; o serviço reúne todo o contexto das mensagens daquele contato na sessão de atendimento atual (o ticket/chamado aberto), consulta uma base de conhecimento originada de um arquivo Excel na pasta `docs/`, gera a melhor resposta usando uma LLM (via OpenRouter) com uma Persona de suporte definida, e devolve essa resposta ao cliente através da API do Digisac. O objetivo desta fatia é entregar o fluxo completo (webhook do Digisac → deduplicação → montagem do contexto do ticket → consulta ao conhecimento → LLM → resposta enviada ao Digisac) ponta a ponta."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Responder uma mensagem de cliente com base em conhecimento relevante (Priority: P1)

Quando uma mensagem de um cliente chega ao sistema (via webhook do Digisac), ela deve
receber uma resposta gerada com base em conteúdo relevante da base de conhecimento (o
Excel em `docs/`) e alinhada ao tom e aos limites definidos para o atendimento —
entregando a promessa central do produto: responder automaticamente, com qualidade, e
devolver a resposta ao cliente pelo próprio Digisac.

**Why this priority**: É o valor central do produto. Sem esse fluxo funcionando, a
integração com o Digisac não tem propósito.

**Independent Test**: Enviar uma mensagem de exemplo (que corresponda a um tópico
presente na base de conhecimento) ao ponto de entrada do webhook e confirmar que uma
resposta é gerada, registrada e entregue de volta ao Digisac, sem intervenção manual.

**Acceptance Scenarios**:

1. **Given** a base de conhecimento (Excel) contém conteúdo relevante a um tópico,
   **When** chega uma mensagem de cliente sobre esse tópico, **Then** o sistema entrega
   uma resposta que reflete esse conteúdo e segue o tom/persona definidos.
2. **Given** a base de conhecimento não contém nenhum conteúdo relevante à mensagem
   recebida, **When** o sistema tenta gerar uma resposta, **Then** o sistema não inventa
   uma resposta arriscada — segue o comportamento definido para esse caso (ver FR-004).

---

### User Story 2 - Responder considerando todo o histórico do ticket aberto (Priority: P1)

Uma conversa de suporte raramente é uma única mensagem isolada. Quando o cliente envia
uma nova mensagem, o sistema deve considerar **todas as mensagens já trocadas naquela
sessão de atendimento** — ou seja, no ticket/chamado atualmente aberto do contato no
Digisac — ao gerar a resposta, para que a resposta faça sentido no contexto do diálogo
(ex.: uma pergunta de acompanhamento como "e no caso do plano anual?").

**Why this priority**: Responder sem o histórico do ticket produz respostas
descontextualizadas que quebram a experiência de atendimento — é parte essencial do
valor central, não um refinamento posterior.

**Independent Test**: Simular um ticket com uma sequência de mensagens do cliente e
confirmar que a resposta à última mensagem leva em conta as mensagens anteriores do
mesmo ticket, e não apenas a última mensagem isolada.

**Acceptance Scenarios**:

1. **Given** um contato tem um ticket aberto com mensagens anteriores, **When** chega
   uma nova mensagem desse contato nesse ticket, **Then** o sistema monta o contexto
   com todas as mensagens desse ticket antes de gerar a resposta.
2. **Given** um contato inicia um novo atendimento sem ticket anterior aberto, **When**
   chega a primeira mensagem, **Then** o sistema trata o contexto como apenas essa
   mensagem, sem exigir histórico prévio.

---

### User Story 3 - Não duplicar resposta quando o mesmo evento é reenviado (Priority: P2)

Plataformas de mensagens (incluindo o Digisac) podem reenviar o mesmo evento de webhook
(por timeout ou retry de rede). O sistema não pode gerar nem entregar duas respostas
para a mesma mensagem original.

**Why this priority**: Duplicidade de resposta é visível ao cliente final e diretamente
prejudicial à experiência — precisa ser validada nesta fatia, não deixada para depois.

**Independent Test**: Enviar o mesmo evento de webhook (mesmo identificador) duas vezes
seguidas e confirmar que apenas uma resposta é gerada e entregue.

**Acceptance Scenarios**:

1. **Given** um evento já foi processado com sucesso, **When** o mesmo evento (mesmo
   identificador único do webhook do Digisac) chega novamente, **Then** o sistema
   reconhece a duplicidade e não reprocessa nem reenvia uma nova resposta.

---

### User Story 4 - Rastrear uma mensagem do início ao fim (Priority: P3)

Um operador precisa conseguir localizar todo o histórico de processamento de uma
mensagem específica (recebida → contexto do ticket montado → conhecimento consultado →
resposta gerada → resposta entregue ao Digisac) usando um único identificador, para
diagnosticar problemas.

**Why this priority**: Importante para operação e depuração, mas não bloqueia a
validação do valor central (User Stories 1 e 2) nem a deduplicação (User Story 3).

**Independent Test**: Processar uma mensagem de ponta a ponta e, usando apenas o
identificador de correlação gerado na entrada, localizar o registro de cada etapa.

**Acceptance Scenarios**:

1. **Given** uma mensagem foi processada (com sucesso ou falha), **When** um operador
   consulta pelo identificador de correlação dessa mensagem, **Then** todas as etapas
   percorridas (recepção, montagem de contexto, consulta ao conhecimento, geração de
   resposta, entrega ao Digisac) são visíveis com esse mesmo identificador.

---

### Edge Cases

- **Mensagem malformada/incompleta** (sem conteúdo de texto — ex.: só mídia/áudio, ou
  um evento do Digisac que não é mensagem de cliente): o sistema deve rejeitar de forma
  controlada, sem quebrar o processamento de outras mensagens, e sem tentar responder.
- **Falha ou lentidão ao montar o contexto do ticket** (consulta ao Digisac/base local
  indisponível): retry limitado com backoff; se persistir, registrar para revisão
  manual em vez de responder sem contexto.
- **Falha ou vazio ao consultar o conhecimento** (Excel indisponível/corrompido, ou
  nenhum trecho relevante): distinguir "indisponível" (erro de infra → retry/registro)
  de "nenhum conteúdo relevante" (erro de negócio → escalonar, ver FR-004).
- **Falha na geração de resposta pela LLM** (timeout, erro, ou saída fora do formato
  esperado): retry limitado, depois registro para revisão manual.
- **Falha na entrega da resposta ao Digisac**: a resposta já gerada não pode ser
  perdida — deve ficar registrada para nova tentativa/revisão manual.
- **Evento duplicado do webhook**: deduplicação estritamente por identificador único do
  evento do Digisac; deduplicação por conteúdo semelhante está fora de escopo.
- **Queda do processo entre o ACK do webhook e a conclusão do processamento**: limite
  conhecido e aceito do MVP (sem fila dedicada) — mitigado pela persistência do payload
  bruto do webhook, que permite replay manual.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: O sistema DEVE aceitar mensagens de clientes através do webhook do Digisac
  e confirmar o recebimento **imediatamente** (ACK < 2s), independentemente de quanto
  tempo o processamento subsequente levar; o processamento ocorre de forma assíncrona,
  fora do caminho síncrono da resposta HTTP ao webhook.
- **FR-002**: O sistema DEVE identificar unicamente cada evento recebido (por um
  identificador único do evento do Digisac) e NÃO DEVE processar o mesmo evento mais de
  uma vez **enquanto o processo estiver em execução** (deduplicação *best-effort* em
  memória — sem banco neste estágio; ver Assumptions). Reentregas do Digisac dentro da
  janela do cache são ignoradas.
- **FR-003**: Antes de gerar a resposta, o sistema DEVE montar o contexto da conversa a
  partir de **todas as mensagens da sessão de atendimento atual do contato** — ou seja,
  o ticket/chamado aberto no Digisac —, e não apenas a última mensagem recebida.
- **FR-004**: O sistema DEVE recuperar informação relevante de uma base de conhecimento
  **originada de um arquivo Excel na pasta `docs/`** para fundamentar a resposta. Quando
  nenhum conteúdo relevante for encontrado, ou quando a resposta não puder ser gerada
  com confiança suficiente, o sistema DEVE preferir **escalonar para atendimento
  humano** em vez de responder com informação não fundamentada. (O critério fino de
  "confiança suficiente" é uma decisão de implementação/plano — ver Assumptions.)
- **FR-005**: O sistema DEVE gerar respostas consistentes com uma persona de suporte
  pré-definida (tom, escopo de atuação, limites do que pode/não pode afirmar), mantida
  em configuração versionada, fora da lógica de infraestrutura.
- **FR-006**: O sistema DEVE entregar a resposta gerada ao cliente **através da API do
  Digisac**, no ticket/conversa correspondente ao contato que originou a mensagem.
- **FR-007**: O sistema DEVE registrar em **logs estruturados** todo evento processado e
  seu resultado (sucesso, escalonamento ou falha), permitindo que um operador revise esse
  histórico depois via agregação de logs (sem banco neste estágio).
- **FR-008**: O sistema DEVE atribuir um identificador único de correlação a cada
  mensagem recebida e propagá-lo por todas as etapas até a entrega da resposta, de forma
  que uma única mensagem possa ser rastreada de ponta a ponta.
- **FR-009**: Cada etapa de processamento (busca de histórico, consulta ao conhecimento,
  geração de resposta, entrega ao Digisac) DEVE ter novas tentativas limitadas com
  backoff; caso a etapa falhe mesmo após as tentativas, o sistema DEVE **registrar a
  falha em log estruturado** (etapa, tipo de erro, `correlationId`) para acompanhamento
  manual, em vez de falhar silenciosamente.
- **FR-010**: O sistema NÃO DEVE registrar o conteúdo das mensagens do cliente em logs
  (apenas IDs/metadados/status). Neste estágio **não há persistência de conteúdo de
  conversa** (sem banco) — logo não há PII em repouso no serviço, o que dispensa job de
  expurgo. O conteúdo trafega apenas em memória e é devolvido ao Digisac.
- **FR-011**: A lógica central de negócio (montagem de contexto, consulta ao
  conhecimento, geração de resposta) NÃO DEVE depender do formato específico do Digisac:
  a integração com o Digisac (entrada via webhook e saída via API) DEVE ficar isolada
  atrás de uma interface de plataforma de suporte, de modo que trocar ou adicionar outra
  plataforma no futuro não exija alterar essa lógica central.

### Key Entities *(include if feature involves data)*

> Neste estágio **não há banco**: as entidades abaixo são **objetos transitórios** (existem
> só em memória durante o processamento), exceto o cache de deduplicação (também em memória,
> porém volátil entre requisições).

- **Conversation**: a sessão de atendimento, mapeada ao **ticket aberto do contato no
  Digisac**; usada em memória para rotear a resposta e o status (aberta/escalada).
- **Message**: uma mensagem, de entrada (do cliente) ou de saída (resposta gerada); trafega
  em memória e é entregue ao Digisac — **não é persistida**.
- **Cache de deduplicação**: conjunto **em memória com TTL** dos IDs de evento já vistos —
  base da deduplicação *best-effort* (volátil; substitui a antiga tabela `ProcessedEvent`).
- **KnowledgeEntry**: item de conhecimento **carregado do arquivo Excel em `docs/`**; sem
  indexação vetorial (sem pgvector) — consulta lexical direta.
- **Registro de falha (log)**: evento de **log estruturado** para cada falha (etapa, tipo de
  erro, `correlationId`) — substitui a antiga tabela `FailureRecord`; sem replay automático.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Para uma mensagem de exemplo cujo tópico existe na base de conhecimento,
  uma resposta completa é gerada e entregue ao Digisac ponta a ponta, sem intervenção
  manual, em até 15 segundos em pelo menos 95% das execuções de teste.
- **SC-002**: Para uma nova mensagem em um ticket com histórico, a resposta gerada leva
  em conta as mensagens anteriores do mesmo ticket em 100% dos casos de teste de
  acompanhamento (perguntas cuja resposta correta depende do histórico).
- **SC-003**: Reenviar o mesmo evento de webhook (mesmo processo em execução) qualquer
  número de vezes produz exatamente uma resposta entregue. *(Limitação conhecida: após
  reinício do processo, o cache de dedup é zerado — reentrega imediatamente após restart
  pode gerar uma segunda resposta; ver Assumptions.)*
- **SC-004**: Para 100% dos eventos processados (com sucesso, escalonamento ou falha), um
  operador consegue localizar o histórico completo de processamento **nos logs** usando
  apenas o `correlationId` da mensagem.
- **SC-005**: Quando qualquer etapa falha mesmo após novas tentativas, 100% das
  mensagens afetadas geram um **registro de falha em log** (etapa + erro + `correlationId`)
  — nenhuma falha silenciosa.

## Assumptions

- A plataforma de suporte integrada é o **Digisac**; a entrada é um webhook do Digisac e
  a saída é a API do Digisac. Os detalhes de autenticação (OAuth2 Bearer), formato exato
  do payload do webhook, endpoint de envio de mensagem e eventual validação de
  assinatura/HMAC são detalhes de implementação a confirmar na fase de plano, a partir da
  documentação oficial do Digisac.
- A base de conhecimento é um **arquivo Excel na pasta `docs/`**, de tamanho pequeno o
  suficiente para ser consultado diretamente no MVP, **sem banco de dados vetorial
  (pgvector)**. A adoção de busca semântica/vetorial é um gatilho de evolução futuro
  (quando a base crescer ou a busca direta deixar de ser suficiente), não parte desta
  fatia.
- "Sessão de atendimento" mapeia para o **ticket/chamado aberto do contato no Digisac**;
  o histórico desse ticket é **buscado ao vivo na API do Digisac** (não há base local) e
  compõe o contexto passado à LLM.
- **Sem persistência / sem banco neste estágio** (decisão de custo): o sistema opera
  apenas com **retries e logs**. A deduplicação é *best-effort* via **cache em memória com
  TTL** (volátil entre reinícios); falhas vão para **logs estruturados** (sem tabela de
  replay). Reintroduzir persistência (ex.: Redis/Postgres) é gatilho de evolução se a
  duplicidade pós-restart ou a perda em crash deixarem de ser aceitáveis.
- **Sem PII em repouso**: como nada de conteúdo de conversa é persistido, a política de
  retenção de 90 dias **não requer job de expurgo** neste estágio; basta manter logs sem
  conteúdo de mensagem.
- **Modelo LLM (via OpenRouter)**: default `google/gemini-3.1-flash-lite` (melhor
  custo-benefício multilíngue/PT-BR + latência), configurável por env; fallback de
  qualidade `openai/gpt-5.4-mini`; opção ultra-econômica `deepseek/deepseek-v4-flash`.
  (Custo do OpenRouter é pay-as-you-go, à parte da infra.)
- **Hospedagem**: container único sempre-ligado (o processamento em background após o ACK
  exige instância viva). Plataforma eleita: **DigitalOcean App Platform (~US$5/mês, só
  infra)**; alternativa Hetzner CX23 (~US$4,3). OpenRouter/Digisac fora da conta de infra.
- Existe (ou é definida minimamente para esta fatia) uma persona de suporte com tom,
  escopo de atuação e política de escalonamento básicos, em configuração versionada.
- O sistema opera em modo **single-tenant** nesta fatia do MVP (`tenantId` presente nos
  objetos para evolução futura, sem isolamento rígido).
- Não existe infraestrutura de fila dedicada; o processamento assíncrono ocorre dentro
  do mesmo processo (in-process). Trade-off aceito: queda do processo entre o ACK e a
  conclusão pode perder a mensagem (sem persistência de fila/payload); o Digisac pode
  reentregar.
- **Critério de escalonamento (FR-004)**: no MVP, o default é escalonar quando nenhum
  conteúdo relevante é encontrado no Excel ou quando a LLM sinaliza que não consegue
  responder com confiança; um critério numérico de confiança mais fino será definido na
  fase de plano ao detalhar a geração de resposta.
- A meta de latência ponta a ponta usada em SC-001 (15 segundos) é a referência já
  definida a nível de projeto, sujeita a validação com as restrições reais do Digisac.
