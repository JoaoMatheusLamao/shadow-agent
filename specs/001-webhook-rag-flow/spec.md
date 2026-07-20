# Feature Specification: Fluxo Ponta a Ponta — Mensagem, Conhecimento e Resposta

**Feature Branch**: `001-webhook-rag-flow`

**Created**: 2026-07-19

**Status**: Draft

**Input**: User description: "Primeiro slice vertical do MVP do middleware de suporte com IA: receber uma mensagem de cliente através de um endpoint de webhook (adapter mock de plataforma de suporte, já que a plataforma real ainda não foi escolhida), recuperar contexto relevante de uma base de conhecimento (RAG via busca vetorial no Postgres/pgvector), gerar uma resposta usando um modelo de LLM acessado via OpenRouter combinado com uma Persona de suporte definida, e devolver essa resposta através de um adapter de saída mock (simulando o callback de resposta da plataforma de suporte). O objetivo deste slice é validar ponta a ponta o fluxo completo (webhook -> dedupe -> RAG -> LLM -> resposta) com dados de exemplo, sem ainda integrar uma plataforma de suporte real."

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Responder uma mensagem de cliente com base em conhecimento relevante (Priority: P1)

Quando uma mensagem de um cliente chega ao sistema, ela deve receber uma resposta
gerada com base em conteúdo relevante da base de conhecimento e alinhada ao tom e
aos limites definidos para o atendimento — validando que a promessa central do
produto (responder automaticamente com qualidade) funciona de ponta a ponta antes
de qualquer integração real com uma plataforma de suporte.

**Why this priority**: É o valor central do produto. Sem esse fluxo funcionando,
nenhuma integração real com uma plataforma de suporte tem propósito.

**Independent Test**: Enviar uma mensagem de exemplo (que corresponda a um tópico
presente na base de conhecimento de teste) para o ponto de entrada do sistema e
confirmar que uma resposta é gerada, registrada e entregue ao destino de saída,
sem qualquer intervenção manual.

**Acceptance Scenarios**:

1. **Given** a base de conhecimento contém conteúdo relevante a um tópico, **When**
   chega uma mensagem de cliente sobre esse tópico, **Then** o sistema entrega uma
   resposta que reflete esse conteúdo e segue o tom/persona definidos.
2. **Given** a base de conhecimento não contém nenhum conteúdo relevante à
   mensagem recebida, **When** o sistema tenta gerar uma resposta, **Then** o
   sistema não inventa uma resposta arriscada — ele segue o comportamento definido
   para esse caso (ver FR-004).

---

### User Story 2 - Não duplicar resposta quando o mesmo evento é reenviado (Priority: P2)

Plataformas de mensagens frequentemente reenviam o mesmo evento (por timeout ou
retry de rede). O sistema não pode gerar nem entregar duas respostas para a mesma
mensagem original.

**Why this priority**: Duplicidade de resposta é visível ao cliente final e
diretamente prejudicial à experiência — precisa ser validada já nesta primeira
fatia, não deixada para depois.

**Independent Test**: Enviar o mesmo evento (mesmo identificador) duas vezes
seguidas e confirmar que apenas uma resposta é gerada e entregue.

**Acceptance Scenarios**:

1. **Given** um evento já foi processado com sucesso, **When** o mesmo evento
   (mesmo identificador único) chega novamente, **Then** o sistema reconhece a
   duplicidade e não reprocessa nem reenvia uma nova resposta.

---

### User Story 3 - Rastrear uma mensagem do início ao fim (Priority: P3)

Um operador precisa conseguir localizar todo o histórico de processamento de uma
mensagem específica (recebida → contexto recuperado → resposta gerada → resposta
entregue) usando um único identificador, para diagnosticar problemas.

**Why this priority**: Importante para operação e depuração, mas não bloqueia a
validação do valor central do produto (User Story 1) nem o comportamento de
deduplicação (User Story 2).

**Independent Test**: Processar uma mensagem de ponta a ponta e, usando apenas o
identificador de correlação gerado na entrada, localizar o registro de cada etapa
do processamento dessa mensagem.

**Acceptance Scenarios**:

1. **Given** uma mensagem foi processada (com sucesso ou falha), **When** um
   operador consulta pelo identificador de correlação dessa mensagem, **Then**
   todas as etapas percorridas por ela (recepção, recuperação de contexto,
   geração de resposta, entrega) são visíveis com esse mesmo identificador.

---

### Edge Cases

- O que acontece quando a mensagem recebida está malformada ou incompleta (sem
  conteúdo de texto, por exemplo)? O sistema deve rejeitar de forma controlada,
  sem quebrar o processamento de outras mensagens.
- O que acontece quando a recuperação de contexto (base de conhecimento) falha ou
  demora além do esperado? O sistema deve tentar novamente um número limitado de
  vezes e, se persistir a falha, registrar a mensagem para revisão manual em vez
  de travar ou perdê-la silenciosamente.
- O que acontece quando a geração de resposta (modelo de linguagem) falha, dá
  timeout, ou retorna algo fora do formato esperado? Mesmo comportamento: retry
  limitado, depois registro para revisão manual.
- O que acontece quando a entrega da resposta ao destino de saída falha? A
  resposta já gerada não pode ser perdida — deve ficar registrada para nova
  tentativa/revisão manual.
- O que acontece se dois eventos com identificadores diferentes, mas que na
  prática representam a mesma mensagem do cliente (ex: reenvio com novo ID),
  chegarem? Fora de escopo desta fatia — a deduplicação aqui é estritamente por
  identificador de evento, não por conteúdo da mensagem.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: O sistema DEVE aceitar mensagens de clientes através de um ponto de
  entrada único e confirmar o recebimento imediatamente, independentemente de
  quanto tempo o processamento subsequente (recuperação de contexto, geração de
  resposta, entrega) levar.
- **FR-002**: O sistema DEVE identificar unicamente cada evento recebido e NÃO
  DEVE processar o mesmo evento mais de uma vez.
- **FR-003**: O sistema DEVE recuperar contexto relevante de uma base de
  conhecimento para cada mensagem recebida, antes de gerar a resposta.
- **FR-004**: Quando nenhum contexto relevante for encontrado, ou quando a
  resposta gerada tiver baixa confiança, o sistema DEVE preferir escalonar para
  atendimento humano em vez de responder [NEEDS CLARIFICATION: qual é o
  critério/sinal usado para decidir que a confiança é baixa o suficiente para
  escalonar em vez de responder, nesta fatia inicial?].
- **FR-005**: O sistema DEVE gerar respostas consistentes com uma persona de
  suporte pré-definida (tom, escopo de atuação, limites do que pode/não pode
  afirmar).
- **FR-006**: O sistema DEVE entregar a resposta gerada através de um canal de
  saída que representa o ponto onde, futuramente, a resposta real seria enviada
  ao cliente.
- **FR-007**: O sistema DEVE registrar todo evento processado e seu resultado
  (sucesso ou falha), permitindo que um operador revise esse histórico depois.
- **FR-008**: O sistema DEVE atribuir um identificador único de correlação a cada
  mensagem recebida e propagá-lo por todas as etapas até a entrega da resposta,
  de forma que uma única mensagem possa ser rastreada de ponta a ponta.
- **FR-009**: Caso a geração de resposta falhe mesmo após novas tentativas, o
  sistema DEVE registrar a falha com detalhe suficiente para acompanhamento
  manual, em vez de perder a mensagem silenciosamente.
- **FR-010**: O sistema NÃO DEVE persistir o conteúdo da mensagem do cliente em
  registros de texto irrestritos/não controlados — o armazenamento do conteúdo
  segue a política de retenção de dados do projeto (90 dias).
- **FR-011**: Como nem a plataforma de suporte real nem a estratégia de
  múltiplos clientes (tenants) foram definidas ainda, os pontos de entrada e
  saída desta fatia DEVEM usar uma implementação substituta/simulada, sem que
  essa escolha afete como o restante do fluxo se comporta — trocar essas
  implementações no futuro não deve exigir mudança na lógica central descrita
  nos requisitos acima.

### Key Entities *(include if feature involves data)*

- **Conversation**: representa a troca contínua entre um cliente e o sistema;
  agrupa mensagens relacionadas e mantém um status (aberta, escalada, encerrada).
- **Message**: uma única mensagem, de entrada (do cliente) ou de saída (resposta
  gerada); carrega o conteúdo e o identificador de correlação.
- **ProcessedEvent**: registro de que um evento de entrada específico já foi
  visto e processado — a base do mecanismo de deduplicação.
- **KnowledgeChunk**: um trecho da base de conhecimento usado para fundamentar a
  resposta gerada.
- **FailureRecord**: registro de uma falha ocorrida em qualquer etapa do
  processamento, com detalhe suficiente para retomada manual.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Para uma mensagem de exemplo cujo tópico existe na base de
  conhecimento de teste, uma resposta completa é gerada e entregue de ponta a
  ponta, sem intervenção manual, em até 15 segundos em pelo menos 95% das
  execuções de teste.
- **SC-002**: Reenviar o mesmo evento de entrada qualquer número de vezes produz
  exatamente uma resposta processada e uma resposta entregue.
- **SC-003**: Para 100% dos eventos processados (com sucesso ou falha), um
  operador consegue localizar o histórico completo de processamento usando
  apenas o identificador de correlação da mensagem.
- **SC-004**: Quando a recuperação de contexto ou a geração de resposta falha
  mesmo após novas tentativas, 100% das mensagens afetadas ficam registradas
  para revisão manual, nenhuma é perdida silenciosamente.

## Assumptions

- Não existe ainda integração real com nenhuma plataforma de suporte; adapters
  substitutos/simulados representam a futura integração real, conforme o plano
  faseado do projeto.
- O conteúdo da base de conhecimento usado para validar esta fatia é um conjunto
  de exemplo pequeno, preparado especificamente para este teste — não é o
  conteúdo de produção em escala real.
- Existe (ou é definida minimamente para esta fatia) uma persona de suporte com
  tom, escopo de atuação e política de escalonamento básicos.
- O sistema opera em modo single-tenant nesta fatia do MVP.
- A política de retenção de dados de conversa (90 dias) já está definida a nível
  de projeto e se aplica a qualquer dado persistido nesta fatia.
- Não existe infraestrutura de fila dedicada disponível; o processamento
  assíncrono ocorre dentro do mesmo processo em execução.
- A meta de latência ponta a ponta usada em SC-001 (15 segundos) é a mesma meta
  de referência já definida a nível de projeto, sujeita a validação futura com
  as restrições reais da plataforma de suporte quando esta for escolhida.
