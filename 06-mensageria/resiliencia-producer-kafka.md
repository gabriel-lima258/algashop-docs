# Resiliência no producer: o publicador que não mente

> Até aqui o catálogo publicava com fé: `send()` assíncrono, nenhum ack exigido, falha virando log. A Fase 41 abre o módulo de resiliência pelo lado de quem escreve — e a pergunta que o organiza é uma só: **o que o produtor pode afirmar depois que o `send()` retorna?** A resposta envolve acks, réplicas em sincronia, idempotência de transporte, uma cadeia de timeouts que sobe até o gateway, e a decisão mais agressiva da fase: amarrar a publicação à transação do banco — falha no Kafka vira rollback no Mongo. Este é o primeiro de três documentos; os irmãos: [resiliência no consumer](./resiliencia-consumer-kafka.md) e [outbox e checkout assíncrono](./outbox-e-checkout-assincrono.md).
> Código real: `product-catalog/.../infrastructure/kafka/KafkaConfig.java`, `application/EventPublishingException.java`, `infrastructure/listener/product/ProductEventListener.java`, `application/product/management/ProductManagementApplicationService.java`, `application-development-env.yml`; meta — `docker-compose.tools.yml` (portas por nó).

---

## acks: o que o broker promete de volta

| `acks` | O produtor considera enviado quando... | O que se perde |
|---|---|---|
| `0` | escreveu no socket | tudo — nem sabe se o broker existe |
| `1` | o **líder** gravou no log dele | o líder cai antes de replicar → mensagem some |
| `all` | as réplicas em sincronia (`min.insync.replicas`) confirmaram | latência — e é o que se paga |

O catálogo usa `acks=all` com o tópico em `min.insync.replicas=2` (3 réplicas): o ack só vem quando **duas** cópias existem. A conta de disponibilidade fecha com o cluster: **1 broker fora** → ISR ainda tem 2, publica normal; **2 brokers fora** → `NotEnoughReplicasException`, o produtor não consegue afirmar durabilidade e **falha alto** — que é exatamente o comportamento desejado.

Atenção à direção da implicação: é o **`enable.idempotence=true` que exige `acks=all`**, não o contrário (e ele é default desde o Kafka 3.0). A idempotência do producer é de **transporte**: o broker deduplica reenvios do próprio producer (producer id + sequence number) durante os retries internos — retry interno deixa de significar mensagem duplicada no log. Não confundir com a idempotência de **negócio** do consumidor ([doc irmão](./resiliencia-consumer-kafka.md)): esta protege contra redelivery, aquela contra retry de rede.

## A cadeia de timeouts — quem engloba quem

```
max.block.ms (5s)          o send() esperando metadados, antes de existir future
  ⊂ request.timeout.ms (5s)   UMA ida ao broker
  ⊂ delivery.timeout.ms (30s) o teto REAL: envio + retries internos + backoff + ack
    ⊂ get(40s)                rede de segurança ACIMA do teto do producer
      ⊂ response-timeout do gateway (60s)   o cliente ainda não desistiu
```

Cada camada é deliberadamente maior que a de dentro: o producer desiste aos 30s e o `get(40)` retorna com a falha **antes** do gateway estourar aos 60s — então dá tempo de fazer rollback e devolver **503** em vez de o cliente receber 504 com a transação pendurada. `retries` fica no default (`Integer.MAX_VALUE`): com idempotence ligado, quem limita é o `delivery.timeout.ms`, não a contagem.

> **A recalibração do gateway — e a lição que quase se perdeu.** O `response-timeout` dos gateways subiu de 5s para 60s (TimeLimiter novo nos dois), porque a rota de escrita do catálogo agora pode legitimamente levar ~35s no pior caso. Mas o `connect-timeout` tinha ido junto para 60s — e **connect não tem motivo**: estabelecer TCP na mesma rede é questão de milissegundos, e 60s ali significa thread presa por um minuto num serviço morto, desfazendo a [calibração da fase de resiliência do gateway](../04-infraestrutura/resiliencia-no-gateway.md). Voltou para 1s. A pendência de lá ficou mais aguda: **timeout por rota** — 60s só na escrita do catálogo; o resto não precisava pagar esse preço.

## get() síncrono: a falha muda de thread — e isso muda tudo

```java
result = kafkaTemplate.send(record).get(40, TimeUnit.SECONDS);
```

A versão assíncrona (`whenComplete`) entrega a falha **em outra thread**: o request já respondeu, o banco já commitou, e o erro vira log que ninguém amarra a nada — dual-write na forma mais silenciosa. O `get()` bloqueante traz a falha para **a thread transacional**, onde ela pode fazer algo: `EventPublishingException` (carregando a **causa** — sem o `super(message, cause)` o stack do broker sumia do log, e é o dado que se quer ao derrubar brokers) sobe até o `MongoTransactionManager`, a sessão aborta, o `ApiExceptionHandler` traduz em **503 Service Unavailable**.

O conjunto que torna isso verdade são **três mudanças combinadas**, nenhuma suficiente sozinha:

1. `@Transactional` em `create/update/disable/enable` (antes só restock/withdraw tinham — o comentário antigo "transação ali não acrescentaria nada" morreu junto);
2. `@Async` **removido** do handler de preço — o handler volta para a sessão transacional;
3. o `get()` lançando na thread do request.

**Isso NÃO é outbox — é transação estendida.** O commit do Mongo passa a esperar o ack do Kafka: prevenção, não compensação. O que se compra: nunca commitar sem evento publicado. O que se paga: a latência do broker dentro do request (~35s no pior caso, com a transação Mongo aberta — o `transactionLifetimeLimitSeconds` default do servidor é 60s, a folga existe mas é finita) e a **janela inversa**: o ack que se perde no timeout deixa um evento **entregue** com a transação **abortada**. O catálogo aceitou essa troca conscientemente; o ordering, com mais eventos em jogo, foi de [outbox de verdade](./outbox-e-checkout-assincrono.md) — os dois desenhos convivem no projeto de propósito, para comparação.

## O cluster reorganizado: uma porta para cada nó

O compose trocou as portas internas repetidas (9090/9091 em todos) por **únicas por nó**:

| nó | PLAINTEXT | CONTROLLER | EXTERNAL (host, 1:1) |
|---|---|---|---|
| kafka-1 | 9081 | 9191 | 9092 |
| kafka-2 | 9082 | 9192 | 9093 |
| kafka-3 | 9083 | 9193 | 9094 |

Funcionalmente o esquema antigo também funcionava (cada container tem seu namespace de rede) — o ganho é de **diagnóstico**, e diagnóstico é o objetivo quando o exercício é `docker stop algashop-kafka-2` e observar: com portas únicas, o `9082` num log de reconexão só pode ser o nó 2, e o mapping 1:1 (`9093:9093`) elimina a tradução mental "9093 do host é o 9092 do nó 2". Healthchecks, Kafka UI e os `application-docker-env` foram junto (os docker-env tinham ficado órfãos apontando o 9090 extinto — o catálogo com `admin.fail-fast` nem subiria no compose).

O cenário completo, do stop ao cliente: **1 broker fora** → producer reconecta no líder novo e segue; **2 fora** → `NotEnoughReplicas` → `EventPublishingException` → rollback do Mongo → 503 — a falha atravessa todas as camadas com nome e sobrenome.

## idempotency-key: a ponta produtora do par

Todo evento nasce com `idempotencyKey` (UUID v7, time-based) que viaja **só no header** `idempotency-key` — `@JsonIgnore` o mantém fora do payload, porque ele não é um fato do domínio, é metadado de entrega. O consumidor usa essa chave na [guarda de idempotência](./resiliencia-consumer-kafka.md). Armadilha de Lombok no caminho: `@Builder` **ignora inicializadores de campo** — sem `@Builder.Default`, um `.builder().build()` sairia com a chave nula e o publisher **silenciosamente** omitiria o header (o guard é `if != null`). Consertado nas 6 classes; o warning de compilação do Lombok era o único aviso.

## Armadilhas encontradas

- **A exceção que engolia a própria causa.** O construtor `(message, event, throwable)` da `EventPublishingException` recebia o `Throwable` e **não o passava ao super** — o log mostraria a exceção de embrulho sem o "caused by" do broker. Em um módulo cujo objetivo é derrubar brokers e ler o que acontece, perder a causa raiz é perder o módulo.
- **As duas `KafkaException`.** O catch importava a do **Spring** (`org.springframework.kafka`); uma `SerializationException` do **Apache** (lançada síncrona no `send`) escaparia do catch — o rollback ainda aconteceria (a exceção sobe pela mesma thread), mas o cliente veria um 500 genérico em vez do 503 com contexto de evento/agregado. O catch agora cobre as duas.
- **Comentário × código nos timeouts.** O javadoc dizia que o `get` devia ser **menor** que o `delivery.timeout.ms`; o código tinha 40s > 30s. O código estava certo (o get é rede de segurança acima do teto) — o comentário mentia. Comentário que contradiz número é pior que nenhum.
- **`connect-timeout` de carona.** Ver acima — o par subiu junto quando só o `response` precisava.

## Pendências registradas

- [ ] **Timeout por rota nos gateways** (`metadata.response-timeout` do SCG) — os 60s valem até para `GET /products`; ficou AGUDA
- [ ] **Outbox no catálogo** — a transação estendida é decisão consciente, mas a janela inversa (ack perdido = evento sem transação) só a outbox fecha
- [ ] **Dimensionar a janela Mongo** — ~35s de sessão aberta contra `transactionLifetimeLimitSeconds=60s`: folga pequena, sem métrica
- [ ] **Testes** — derrubar broker é exercício manual; um teste com Testcontainers parando o broker documentaria o comportamento

## Checklist de revisão

- [ ] Sei o que cada nível de `acks` permite AFIRMAR — e por que `all` + `min.insync=2` casa com o cluster de 3
- [ ] Sei a direção da implicação idempotence → acks=all, e a diferença transporte × negócio
- [ ] Consigo desenhar a cadeia de timeouts e explicar por que cada camada é maior que a de dentro
- [ ] Sei o que o `get()` muda em relação ao `whenComplete` — e o que é "transação estendida" vs outbox
- [ ] Sei qual janela a transação estendida fecha e qual ela ABRE
- [ ] Derrubando 1 e 2 brokers, sei prever o comportamento ponta a ponta

## Referências

- [Kafka — Producer Configs](https://kafka.apache.org/documentation/#producerconfigs) (acks, delivery.timeout, enable.idempotence)
- [Confluent — Idempotent Producer](https://developer.confluent.io/courses/architecture/broker/#idempotent-producer)
- [Resiliência no consumer](./resiliencia-consumer-kafka.md) · [Outbox e checkout assíncrono](./outbox-e-checkout-assincrono.md) · [Kafka na prática](./kafka-na-pratica.md) · [Resiliência no gateway](../04-infraestrutura/resiliencia-no-gateway.md) · [Transações no Mongo](../02-persistencia/transacoes-mongo.md)
