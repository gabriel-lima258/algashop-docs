# Diário de bordo da mensageria

> **Em uma frase:** o caderno de bastidores da trilha de mensageria — revisões, bugs encontrados e corrigidos, pendências e testes de cada etapa, que saíram dos docs didáticos para eles ficarem leves.

⬅️ Anterior: [Saga orquestrada — robustez](./16-saga-orquestrada-robustez.md) · 🧭 Trilha: [README](../README.md#trilha-de-leitura)

**Como usar:** este doc não é para ler em sequência. Leia primeiro o doc didático (01–16) e venha aqui quando quiser saber *o que deu errado no caminho*, *o que ficou pendente* ou *como aquilo foi testado*. Cada seção tem o número do doc correspondente.

- [Divergências entre docs antigos e código (encontradas na reescrita)](#divergências-entre-docs-antigos-e-código-encontradas-na-reescrita)
- [01](#01--fundamentos-de-arquitetura-orientada-a-eventos-eda) · [02](#02--kafka-a-fundo) · [03](#03--kafka-na-prática-o-primeiro-evento-entre-serviços) · [04](#04--ecst-e-validação-de-eventos-o-consumidor-reage) · [05](#05--contratos-com-asyncapi) · [06](#06--resiliência-no-producer) · [07](#07--resiliência-no-consumer) · [08](#08--outbox) · [09](#09--workflow-coreografado) · [10](#10--saga-coreografada) · [11](#11--saga-orquestrada-conceito-e-mapa) · [12](#12--saga-orquestrada-preparando-o-terreno) · [13](#13--saga-orquestrada-estado-e-persistência) · [14](#14--saga-orquestrada-comandos-e-respostas) · [15](#15--saga-orquestrada-compensações) · [16](#16--saga-orquestrada-robustez)

## Divergências entre docs antigos e código (encontradas na reescrita)

Ao reescrever, os docs passaram a seguir o **código atual**. Estes pontos mostram onde a documentação antiga (ou um comentário no código) tinha ficado para trás — vários são comentários desatualizados que valem uma correção futura:

- [ ] A interface citada como `IntegrationEvent` hoje se chama `OutboundIntegrationEvent` (catálogo).
- [ ] Portas internas do Kafka: o compose usa `algashop-kafka-N:9081/9082/9083` (os docs antigos diziam 9090); as externas seguem `localhost:9092/9093/9094`.
- [ ] `KafkaProductIntegrationEventListener` (ordering) usa SpEL em `topics` e `id`; o `id` define o grupo `ordering.product-events`, embora `group-id: ordering` continue no YAML.
- [ ] ECST de preço: o código usa `ShoppingCartProductAdjustmentService` (implementado pelo `ShoppingCartUpdateProvider`, UPDATE em massa) — o doc antigo dizia que o ajuste passava pelo agregado e que o bulk era código morto; o comentário do service está desatualizado.
- [ ] No handler V2 com guarda de idempotência, o evict do cache acontece **antes** do `refreshProductPrice` (dentro do `runOnce`).
- [ ] `ProductEventListener` (catálogo) usa `@TransactionalEventListener` + `@Async` (after-commit), mas o comentário do topo ainda diz que os handlers são síncronos; o `ProductManagementApplicationService` guarda o comentário antigo "transação ali não acrescentaria nada". O handler de `ProductAddedEvent` ainda tem `@Async`.
- [ ] O `KafkaConfig` do catálogo captura só `org.springframework.kafka.KafkaException` (os docs antigos diziam que cobria também a do Apache).
- [ ] Não-retentáveis do ordering: a lista real é `DomainException`, `DomainEntityNotFoundException`, `ConstraintViolationException`, `DataIntegrityViolationException`, `IllegalArgumentException`, `SagaTransitionException` — o comentário do `KafkaConfig` fala em `BusinessException`.
- [ ] `KafkaBeanValidationConfigurer` do ordering: comentário desatualizado ("10 tentativas sem backoff... sem DLQ").
- [ ] `orderEventsDlt` (`ordering.dlt.ordering.order.events`) continua no `KafkaConfig` do ordering, para um tópico que o ordering só produz.
- [ ] `OutboxRecorder` ainda não valida o payload antes de gravar.
- [ ] `InvoiceManagementApplicationService.refundByOrderId` (billing) continua sem `save()` — grava por dirty checking, mas não publica o `InvoiceCanceledEvent`.
- [ ] `KafkaInvoiceIntegrationCommandListener` (billing): o comentário do handler de `CancelInvoice` aparece duas vezes, e o cabeçalho da classe ainda diz que "a resposta ainda não é enviada".
- [ ] Idempotência dos replies no orquestrador ainda usa checagens por estado (`isReservingStock() || hasSucceeded()`, `isCompensated()`, `wasStockRejected()`), não o "passo esperado".
- Nomes de property: o catálogo usa `stock-commands-topic-name`; o ordering chama o mesmo tópico de `product-catalog-commands-topic-name`.

## 01 — Fundamentos de arquitetura orientada a eventos (EDA)

### Revisões e correções
- O doc original era o estudo conceitual que abriu o módulo (antes de qualquer broker no compose). Citava como código já existente: `AbstractAggregateRoot` no ordering e no catálogo, os listeners `@Async` in-process, a categoria desnormalizada e o `SKIP LOCKED` do billing-scheduler.
- Retrofit (Fase 39): ECST atravessou a fronteira — `ProductPriceChangedV2IntegrationEvent` carrega os preços pelo Kafka e o ordering atualiza carrinhos sem consultar o catálogo; Listed/Delisted fazem notification no mesmo listener.
- A previsão original era que o fluxo pedido → fatura nasceria coreografado; a prática coreografada (pedido → fatura → pagamento → estoque) ficou no doc de workflow coreografado.

### Pendências
- [x] Escolher o broker pelo papel — Kafka (Fase 38): consumidores futuros dos eventos de catálogo precisam do passado, e a expansão retroativa só o log dá.
- [x] Primeiro evento a atravessar a fronteira — previsto `pedido confirmado` (ordering → billing); na Fase 38 atravessou outro par: produto listado/deslistado (catálogo → ordering), key = id do produto. O `pedido confirmado` continuou no mapa.
- [x] Outbox no produtor (pendência da Fase 12) — paga na Fase 41: outbox completa no ordering (recorder transacional + polling publisher); o catálogo com o desenho alternativo (transação estendida). Command Message e Listen to Yourself entraram junto.
- [ ] Contrato de evento versionado — o equivalente de mensageria do Spring Cloud Contract, com teste que o trave.
- [x] DLQ e retentativa — fecharam na Fase 41 (backoff exponencial + DLQ, doc de resiliência no consumer); **reconciliação segue aberta**.
- [ ] Correlação ponta a ponta — o id que conta a história de um pedido através dos serviços.

### Testes
- (nenhum registro de testes neste doc)

### Checklist de revisão (original)
- O nome do evento é um fato no passado ou um comando disfarçado?
- O evento é de domínio (interno) ou de integração (contrato público)? Quem quebra se ele mudar?
- Notificação magra ou estado carregado — quantos consumidores vão "voltar para perguntar"?
- O canal precisa de entrega única (trabalho) ou cópia por assinante (fato)?
- Um consumidor futuro precisará do passado? Então é log, não fila.
- O fluxo tem compensação? Então a conversa é orquestração/saga.
- Como o evento sobrevive ao crash entre o commit e o publish?

### Referências (original)
- Martin Fowler — What do you mean by "Event-Driven"? (https://martinfowler.com/articles/201701-event-driven.html); Event Sourcing (https://martinfowler.com/eaaDev/EventSourcing.html)
- Enterprise Integration Patterns — Hohpe & Woolf; Kafka Introduction; RabbitMQ Concepts
- Microservices.io — Saga; Transactional Outbox

## 02 — Kafka a fundo

### Revisões e correções
- O doc original era conceitual ("o Kafka ainda não existe no projeto"). Retrofit (Fase 38): o broker chegou, documentado em Kafka na prática.
- Previsão "KRaft em modo combinado, um processo" — fechada de outro jeito: modo combinado, porém com **3 nós**. O formato mais pesado permitiu replicação 3 e `min.insync.replicas=2` reais, com failover observável ao derrubar um broker.
- Previsão "primeiro tópico `pedido confirmado` com key = id do pedido" — o rumo mudou: o primeiro foi `product-catalog.product.events` com key = id do produto (catálogo → ordering). O princípio (key = entidade cuja cronologia importa) se manteve.
- Serialização e contrato do record — meio fechada: JSON via `JacksonJsonSerializer`, com `__TypeId__` lógico desacoplando pacotes; o rigor do Spring Cloud Contract não chegou à mensageria (contrato = cópia manual de POJO, sem teste).
- Partições do primeiro tópico — fechada: 3 partições, olhando o crescimento futuro do grupo `ordering`.

### Pendências
- [ ] Monitorar consumer lag desde o primeiro consumidor — o grupo existe, a Kafka UI (9084) mostra o lag, nenhuma métrica o coleta.

### Testes
- (nenhum registro de testes neste doc)

### Checklist de revisão (original)
- A key escolhida dá a cronologia de quê? É a entidade certa?
- Quantas partições = quanto paralelismo futuro?
- O consumidor novo precisa do passado? A retention cobre?
- O grupo é de trabalho ou de assinatura? São grupos distintos?
- O que acontece durante um rebalance e com que frequência ocorre?
- Quem olha o consumer lag e a partir de que valor acorda alguém?

### Referências (original)
- Kafka — Design; Kafka — KRaft; Confluent — Consumer Groups; Jay Kreps — The Log: What every software engineer should know.

## 03 — Kafka na prática: o primeiro evento entre serviços

### Revisões e correções
- Rumo previsto era `pedido confirmado` (ordering → billing); o curso começou pelo par catálogo → ordering.
- Na época, dos 7 eventos do `Product` só 3 saíam como integração; os outros 4 tinham o `send` comentado à espera de consumidor. (Hoje o `ProductEventListener` publica mais eventos — Added, PriceChangedV2, PlacedOnSale, Listed, Delisted, Restocked etc.)
- **Nome de bean dentro de SpEL é contrato implícito** (terceira aparição da lição; o `@securityChecks` do billing pagou primeiro). O tópico do `@KafkaListener` começou como `#{algaShopMessagingKafkaProperties.productEventTopicName}`; rename da classe muda o nome do bean e quebra em runtime. O doc registrava a troca para placeholder `${algashop.messaging.kafka.product-event-topic-name}`. **Observação da reescrita:** o código atual do `KafkaProductIntegrationEventListener` voltou a usar SpEL (`topics = "#{algaShopMessagingKafkaProperties.productEventTopicName}"`, `id = "#{algaShopMessagingKafkaProperties.productEventsConsumerGroup}"`).
- **Typo quase virou contrato:** o campo saiu como `deslistedAt` no evento de integração dos dois lados; consertado antes do primeiro deploy. O typo continua no evento de domínio, e como o `ModelMapper` do catálogo é `STRICT`, o de-para `deslistedAt → delistedAt` precisou ser explícito no `ModelMapperConfig` — senão o campo iria nulo no JSON sem erro (mesma lição silenciosa do `paymentMethod` no billing).
- A cópia manual do POJO não tem rede: campo renomeado no produtor vira `null` no consumidor sem teste falhar.
- Infra na época: listener interno anunciado em `algashop-kafka-N:9090`; healthcheck via `/dev/tcp` no 9090. **Hoje** (desde a Fase 41) cada nó tem porta interna única 9081/9082/9083, para o log de reconexão dizer qual broker caiu. Os hostnames `algashop-kafka-N` foram adicionados a `etc/hostnames/hostnames` para o cenário host.
- O que NÃO existia na época (Fase 38): outbox (o `ProductEventListener` rodava síncrono após o `save()`, dual-write exposto); retry/DLQ no consumidor; efeito de negócio (handlers só logavam); testes (zero — `spring-boot-starter-kafka-test` no `build.gradle` do catálogo sem uso); métrica de consumer lag.

### Pendências
- [x] Outbox no catálogo — resolvida na Fase 41 por outro caminho: transação estendida (send síncrono dentro do `@Transactional`; falha no Kafka = rollback no Mongo). Outbox de verdade ficou no ordering. Os dois desenhos convivem de propósito, para comparação.
- [x] Retry com backoff + DLQ no listener do ordering — Fase 41: `DefaultErrorHandler` com backoff exponencial (2s/4s/8s), exceções de negócio sem retry, `DeadLetterPublishingRecoverer` publicando em `ordering.dlt.*` na mesma partição.
- [x] Reagir ao evento no ordering — fechada pela metade na Fase 39 (carrinhos e cache), idempotência veio depois (Fase 41).
- [ ] Teste com `spring-kafka-test` — produzir → consumir num broker embarcado.
- [ ] Contrato de mensageria com teste — metade fechada na Fase 40 (AsyncAPI escrito: payload, `__TypeId__`, key, topologia); o teste que o impede de mentir segue pendente.
- [ ] Métrica de consumer lag — sair da UI e entrar no actuator/alerta.

### Testes
- Na Fase 38: zero testes de mensageria; starter de teste presente e sem uso.

### Checklist de revisão (original)
- Explicar por que o evento nasce duas vezes e por que são duas portas.
- O que a key = `productId` garante e o que não garante.
- O caminho do `__TypeId__`: nome lógico, `type.mapping` de cada lado, fallback `ObjectNode`.
- Por que `ProductAdded` não está mapeado no consumidor.
- Por que o perfil docker sobrescreve o bootstrap.
- Onde acontece o dual-write e o que a outbox mudaria.

## 04 — ECST e validação de eventos: o consumidor reage

### Revisões e correções
- **Correção do doc anterior:** a Fase 38 dizia "exceção num handler vira retentativa infinita". Errado — o `DefaultErrorHandler` padrão faz `FixedBackOff(0, 9)`: 10 tentativas sem intervalo e depois **descarta commitando o offset**. Perda silenciosa, sem DLQ. (Resolvido na Fase 41.)
- **Validação com condição invertida:** o `BeanValidationUtil` saiu com `if (violations.isEmpty()) throw` — todo evento válido explodia e o inválido passava. Como o `validate()` está no publisher genérico, eventos sem anotação (Added/Listed/Delisted) lançariam sempre, e como seus handlers eram síncronos, o POST/PUT de produto devolveria 500. Corrigido para `if (!violations.isEmpty())`. Lição: validação só existe com teste do caminho triste.
- **Total que ninguém recalculava (bug de dinheiro):** `ShoppingCart.changeItemPrice` atualizava o total do item mas não chamava `recalculateTotals()` do carrinho (os irmãos `refreshItem` e `changeQuantity` chamam). Corrigido. Junto: `ShoppingCartItem.changeItemPrice` estava `public` (irmãos package-private), permitindo furar o agregado.
- **Código pronto no banco de reservas:** já existia `ShoppingCartProductAdjustmentService`/`ShoppingCartUpdateProvider` (2 UPDATEs JPQL numa transação, recalculando o total no banco, com teste de integração). Na Fase 39 o fluxo novo reimplementou via agregado (carrega → muda → salva), deixando o bulk como código morto — escolha mantida de propósito para exercitar invariantes, ao custo de N+1 (risco de estourar `max.poll.interval.ms` e provocar rebalance). **Observação da reescrita:** o código atual de `ShoppingCartManagementApplicationService.refreshProductPrice`/`changeProductAvailability` chama `shoppingCartProductAdjustmentService` (implementado por `ShoppingCartUpdateProvider`), ou seja, o caminho bulk voltou a ser o usado; o comentário no service ainda descreve o caminho via agregado.
- **Transação única:** sem `@Transactional` nos dois métodos novos do service, cada `add()` abria a própria transação e uma falha no meio deixava metade dos carrinhos atualizada.
- **Evict depois da atualização** (evitar repovoar cache com dado velho). **Observação da reescrita:** no handler do V2 sob guarda de idempotência, o código atual faz o evict antes do `refreshProductPrice` dentro do `runOnce` (comentário "redis nao tem rollback"); nos handlers de notification o evict segue depois.
- `ProductCacheManager` usa API programática (`cache.evictIfPresent`), que não passa pelo `CacheInterceptor` — o `ResilienceCacheErrorHandler` não protege; o try/catch no evict cumpre esse papel.
- `cache.md` revogado na premissa "sobra o TTL": o serviço agora fica sabendo via Kafka; TTL curto virou segunda linha de defesa.
- Retrofit (Fase 40): contratos declarados em AsyncAPI; o primeiro YAML nasceu com o `__TypeId__` do V2 errado e nada acusou (teste contrato × código segue pendente).
- O que não existia na Fase 39: idempotência e guarda de ordem (`changedAt`/`oldSalePrice` ignorados); retry/DLQ; outbox no produtor (handler de preço era `@Async`, falha de publicação nem chegava ao chamador); `production-env` sem `type.mapping` (pré-existente; sem ele o `__TypeId__` sairia como FQCN).

### Pendências
- [x] Idempotência no consumo de preço — Fase 41: guarda por chave (INSERT-first na PK de `processed_message`, mesma transação do efeito), mais geral que compare-and-set.
- [x] Retry com backoff + DLQ — Fase 41, descarte silencioso acabou.
- [x] Outbox no catálogo — Fase 41: catálogo foi de transação estendida (o `@Async` do preço caiu; falha na publicação vira rollback); outbox de verdade nasceu no ordering.
- [ ] Testes: caminho triste da validação (dois lados), handler → carrinho → total recalculado, e decidir o destino do `ShoppingCartUpdateProviderIT`.
- [ ] Trocar para o caminho bulk quando o N+1 doer (ver observação acima: o código atual já parece usar o bulk).
- [ ] Paginar/limitar `findAllContainingItem` — carregava todos os carrinhos afetados de uma vez.
- [ ] `production-env` sem `type.mapping` (registrado como pré-existente).

### Testes
- Na Fase 39: zero testes para validação, handler novo, `findAllContainingItem` e métodos novos do agregado. `ShoppingCartUpdateProviderIT` testava um caminho que, na época, ninguém mais chamava.

### Checklist de revisão (original)
- Apontar no mesmo listener qual handler é notification e qual é ECST, e por quê.
- Por que o V2 nasceu V2 e onde a versão viaja.
- O que `@Valid` no `@Payload` faz sem o `KafkaListenerConfigurer` (nada).
- O destino real do evento que falha (10 tentativas → descarte com offset commitado).
- Por que o evict vem depois e por que a API programática não é protegida pelo `CacheErrorHandler`.
- Defender agregado × bulk e dizer quando trocar.

## 05 — Contratos com AsyncAPI

### Revisões e correções

- **Fase 40**: `docs/asyncapi/` nasce descrevendo `product-catalog.product.events` em AsyncAPI 3.1.0 (`product-catalog.yml` produtor, `ordering.yml` consumidor, `messages.yml` compartilhado). Critério de pronto: os dois contratos passavam no `asyncapi validate`.
- **O `__TypeId__` do V2 saiu errado** no primeiro rascunho (`ProductPriceChangedEventV2...` em vez de `ProductPriceChangedV2...`). O YAML era estruturalmente perfeito, o `validate` não pegou; quem implementasse um consumidor pelo contrato cairia no handler default. Corrigido.
- **Mensagens reais fora do contrato**: `ProductListed`/`ProductDelisted`, publicadas desde a Fase 38 e consumidas com efeito desde a 39, não estavam em nenhum dos documentos. Corrigido.
- **Copy/paste entre perspectivas**: o `ordering.yml` saiu com `summary: "Publishes product events"` numa operação `receive` e listando o V1 que ele não consome. Corrigido.
- **Drift previsto aconteceu (Fase 41)**: o ordering ganhou `ordering.order.commands`, `ordering.order.events` e três DLTs sem contrato. Na Fase 44 o `ordering.order.commands` saiu do código e do contrato junto com o checkout assíncrono (canais, operações e mensagens do comando removidos de `ordering.yml` e `messages.yml`; o schema `OrderSnapshot` ficou sem `shoppingCartId`). Hoje os contratos cobrem eventos de pedido, comandos/respostas da saga, DLTs e há um `billing.yml`.

### Pendências

- [ ] Travar o contrato no CI: `asyncapi validate` no pipeline (barato) e um teste comparando YAML × código (consts do `__TypeId__` × `type.mapping`, `required` × campos dos POJOs), o equivalente do SCC no REST.
- [ ] Gerar o HTML do contrato no pipeline a cada mudança.
- [ ] Declarar o destino do `ProductAdded` (nenhum contrato `receive` o lista; quando tiver consumidor, o contrato do serviço novo nasce junto).
- [ ] `correlationId` nas mensagens / correlação ponta a ponta (pendente desde os fundamentos de EDA; hoje existe `correlationIds.CorrelationId` em `kafka_correlationId` para a saga).
- [ ] Avaliar schema registry/Avro quando os contratos multiplicarem.

### Testes

- Comandos usados: `npx -y @asyncapi/cli validate docs/asyncapi/product-catalog.yml`, `npx -y @asyncapi/cli validate docs/asyncapi/ordering.yml`, `npx -y @asyncapi/cli generate fromTemplate docs/asyncapi/product-catalog.yml @asyncapi/html-template@3.0.0 --use-new-generator -o build/asyncapi-docs`.

## 06 — Resiliência no producer

### Revisões e correções

- **`EventPublishingException` engolia a causa**: o construtor `(message, event, throwable)` não passava o `Throwable` ao `super`; o log perdia o "caused by" do broker. Corrigido.
- **As duas `KafkaException`**: o catch importava só a do Spring (`org.springframework.kafka`); uma `SerializationException` do Apache (lançada síncrona no `send`) escaparia: rollback ainda ocorria, mas o cliente veria 500 genérico em vez de 503. A fonte diz "o catch agora cobre as duas" — **no código atual do `KafkaConfig` do catálogo o catch lista só `org.springframework.kafka.KafkaException`** (ver inconsistências).
- **Comentário × código nos timeouts**: o javadoc dizia que o `get` devia ser menor que o `delivery.timeout.ms`; o código tinha 40s > 30s. O código estava certo; o comentário foi corrigido.
- **`connect-timeout` de carona**: ao subir o `response-timeout` dos gateways de 5s para 60s (TimeLimiter novo nos dois), o `connect-timeout` foi junto para 60s; voltou para 1s.
- **`@Builder` ignora inicializador**: sem `@Builder.Default`, `.builder().build()` saía com `idempotencyKey` nulo e o publisher omitia o header em silêncio (guard `if != null`). Corrigido nas 6 classes de evento; o warning do Lombok era o único aviso.
- **Portas do cluster**: o compose trocou as portas internas repetidas (9090/9091) por únicas por nó (9081-9083 / 9191-9193 / 9092-9094). Healthchecks, Kafka UI e os `application-docker-env` acompanharam (os docker-env estavam órfãos apontando o 9090 extinto — o catálogo com `admin.fail-fast` nem subiria no compose).
- Três mudanças da Fase 41 no catálogo: `@Transactional` em create/update/disable/enable (o comentário antigo "transação ali não acrescentaria nada" morreu — mas ainda há resquício dele no `ProductManagementApplicationService`), `@Async` removido do handler de preço, `get()` lançando na thread do request.
- **Depois (módulo 7, workflow coreografado)**: commit `7f92c34` no product-catalog passou os handlers do `ProductEventListener` a `@TransactionalEventListener` + `@Async` (after-commit), para evitar evento fantasma.

### Pendências

- [ ] Timeout por rota nos gateways (`metadata.response-timeout` do SCG): os 60s valem até para `GET /products`.
- [ ] Outbox no catálogo: só ela fecha a janela inversa (e, com after-commit, a janela "commit sem evento").
- [ ] Dimensionar a janela Mongo: ~35s de sessão aberta contra `transactionLifetimeLimitSeconds=60s`, sem métrica.
- [ ] Teste com Testcontainers parando o broker (hoje derrubar broker é exercício manual).

### Testes

- Roteiro manual: `docker stop algashop-kafka-2` → producer reconecta no líder novo e segue; parar 2 brokers → `NotEnoughReplicas` → `EventPublishingException` → rollback do Mongo → 503.

## 07 — Resiliência no consumer

### Revisões e correções

- **"Retry infinito" era mito**: a Fase 39 documentou retry infinito, mas o default era descarte após 10 tentativas sem intervalo. O comentário novo dizia "3 tentativas"; eram 3 retentativas (4 entregas). Corrigido no comentário do `KafkaConfig`.
- **DLT de tópico sem consumidor**: `ordering.dlt.ordering.order.events` existe, mas o ordering só produz `ordering.order.events`; criado por simetria (o bean `orderEventsDlt` ainda existe no `KafkaConfig` do ordering).
- **Header obrigatório**: `@Header` sem `required=false` fazia mensagem sem `idempotency-key` falhar antes do handler (4 entregas e DLT). Corrigido com fallback "processa sem guarda, com aviso".
- Listeners ganharam `id` próprio (`ordering.product-events`, na época também `ordering.order-commands`), que vira o `group.id`, substituindo o `group-id: ordering` global.
- No código atual a lista de não-retentáveis do ordering é `DomainException`, `DomainEntityNotFoundException`, `ConstraintViolationException`, `DataIntegrityViolationException`, `IllegalArgumentException`, `SagaTransitionException` (a fonte e o comentário do `KafkaConfig` falam em `BusinessException`).

### Pendências

- [ ] Quem olha o DLT? 30 dias de retenção sem consumidor, alerta ou rotina de replay.
- [ ] Template de `byte[]` para o `DeadLetterPublishingRecoverer` (republicar bytes crus em falha de desserialização).
- [ ] Métrica de consumer lag (pendência mais antiga do módulo), agora com DLTs para monitorar.
- [ ] Reverter os tunings de laboratório (`max.poll.records: 1`, `logging.level org.apache.kafka: OFF`).
- [ ] IT com broker embarcado provando retry → DLT e duplicata → skip.

### Testes

- (nenhum número registrado na fonte)

## 08 — Outbox

### Revisões e correções

- **Starvation do `findBatch`**: o original era `SELECT ... ORDER BY m.id` sem `WHERE`. Mensagens com `failedAt` nunca são deletadas e, sendo as mais antigas (UUIDv7), eram sempre as primeiras do lote; com 100 mortas/em backoff, nada novo saía. Corrigido com filtro **por aggregate** (`aggregateId NOT IN (... failedAt IS NOT NULL OR nextAttemptAt > :now)`), mantendo as guardas em memória como segunda camada. Filtrar só a linha quebraria a ordem.
- **Duas chaves de idempotência**: no caminho direto o header vem do campo `idempotencyKey`; no caminho outbox vem do id da linha (o campo do comando nem é persistido, `@JsonIgnore`). Ambas estáveis, mas desligar a outbox muda a identidade das mensagens. Registrado como decisão a unificar.
- **Fase 44**: checkout assíncrono v2 removido (`MyOrderV2Controller`, `OrderAcceptedOutput`, `CheckoutAsyncApplicationService`, `BuyNowAsyncApplicationService`, `ProcessAcceptedCheckoutApplicationService`, portas `ForBuyingWithShoppingCartAsync`/`ForBuyingProductAsync`/`ForProcessingCheckoutAccepted`, `ProcessAcceptedCheckoutIntegrationCommand`, `KafkaOrderIntegrationCommandListener`, `KafkaOrderIntegrationCommandPublisher`, `OutboxOrderIntegrationCommandPublisher`, `ForPublishingOrderIntegrationCommands`, beans de `ordering.order.commands` e DLT, propriedades `orderCommandTopicName`/`orderCommandsConsumerGroup`, alias `Ordering.ProcessAcceptedCheckoutIntegrationCommand`, `OrderSnapshot.shoppingCartId`, `toSnapshot(Order, UUID)` e `toDomain`). Detalhado no doc 12.
- **Outbox para comandos (V011/V012)**: `event_type` → `message_type` (campo `OutboxMessage.messageType`, método `readMessageType`); colunas `correlation_id` e `reply_topic`; `OutboxRecorder.record(channelName, aggregateId, message)` virou `record(OutboxDraft)`. Corrigido o comentário do `OutboxKafkaSender` que ainda dizia `__TypeId__ = eventType`.

### Pendências (da época do checkout assíncrono; várias perderam o objeto com a remoção do v2)

- [ ] Notificar pedidos V2 (`Order.existing()` não emitia eventos de domínio; o `OrderEventListener` nunca disparava para pedidos assíncronos) — obsoleta com a remoção do v2.
- [ ] Deduplicar checkout por carrinho (janela do duplo-POST entre o 202 e o consumo) — obsoleta com a remoção do v2.
- [ ] Replay do outbox: rotina/endpoint para `failedAt`, e backoff exponencial no lugar do fixo (10 × 5s ≈ 50s de tolerância).
- [ ] Consumidor para `ordering.order.events` (billing) e contrato AsyncAPI dos tópicos novos — hoje o contrato existe.
- [ ] Validar no `OutboxRecorder` (o `BeanValidationUtil` só roda nos publishers diretos).
- [ ] Unificar a fonte da `idempotency-key` (id da linha × campo do evento/comando).
- [ ] Latência de 1 minuto (`poll-interval: PT1M`): poll mais curto ou notificação do dispatcher.
- [ ] Testes: outbox grava na transação (MANDATORY estoura fora), dispatcher preserva ordem por aggregate, starvation não volta.

### Testes

- Na época: zero testes para outbox, dispatcher, idempotência e checkout assíncrono.
- Fase 44 (outbox para comandos): `./gradlew check` do ordering verde (304 unitários, 12 de contrato, 185 de integração); o `CheckoutApplicationServiceIT` exercita a gravação com as colunas da V011/V012; envio com os headers novos sem teste (faltaria um IT do `OutboxKafkaSender`).
- Roteiro da remoção do v2: `POST /api/v2/customers/me/orders` com `Content-Type: application/vnd.order-with-shopping-cart.v2+json`, pelo gateway e direto, devolve 404; nenhuma mensagem nova em `ordering.order.commands`; conferir que a tabela `outbox` não tem linhas pendentes para `ordering.order.commands` e que os tópicos antigos continuam no broker até exclusão manual.

## 09 — Workflow coreografado

### Revisões e correções

- **Status do mapa no código (módulo 7–8, histórico):** `OrderPlaced` (aulas 7.4–7.6), `InvoiceIssued` (7.7–7.8), `InvoicePaid` (7.8–7.10), `OrderPaid` (publicado na 7.10, consumido no módulo 8, aulas 8.6–8.8), `StockReservationConfirmed` (8.9–8.10), `OrderReady` (8.10), `InvoiceCanceled` (8.3), `StockReservationRejected` (8.11). Tudo removido depois na Fase 44 (saga orquestrada).
- **Bug do webhook anônimo (aula 7.8):** o webhook FastPay é `permitAll`; ao passar a gravar a fatura, o `AuditorAware` chamou `getAuthenticatedUserId()` e quebrou, porque a requisição chega com `AnonymousAuthenticationToken` e `isAuthenticated()` é true. Correção: `isAnonymous()` no `SecurityCheckApplicationService` do billing e o `auditorProvider` devolvendo `Optional.empty()` também nesse caso (`!isAuthenticated() || isMachineAuthenticated() || isAnonymous()`).
- **Aula 7.9 (infra de consumidor no ordering):** tópico/grupo `billing.invoice.events` / `ordering.invoice-events` em `application-development-env.yaml`; os três `id` de `@KafkaListener` (`productEvents`, `orderCommands`, `invoiceEvents`) saíram do código para `AlgaShopMessagingKafkaProperties` via SpEL; DLT `KafkaConfig.invoiceEventsDlt()` → `ordering.dlt.billing.invoice.events`; `DomainEntityNotFoundException` entrou nos não-retryable; type mapping só `Billing.InvoicePaidIntegrationEvent`; cópia do evento em `core/application/invoice/event/InvoicePaidIntegrationEvent.java`.
- **Aula 7.3:** contrato `billing.yml` declarava `consumeOrderPlacedEvents` (`billing.order-events`), `publishInvoiceEvents`, `consumeInvoiceIssuedEvents` (`billing.invoice-events`), `consumeStockReservationRejectedEvents` (`billing.stock-events`).
- **Aulas 7.5–7.6:** `KafkaOrderIntegrationEventListener` do billing com concurrency 3, retry 2s/4s/8s → `billing.dlt.ordering.order.events`; type mapping `Ordering.OrderPlacedIntegrationEvent:com.algaworks.algashop.billing.application.order.event.OrderPlacedIntegrationEvent`; `GenerateInvoiceInputAssembler` (item vira `"Produto - x2"`, frete vira linha "Shipping" só se > 0, `firstName`+`lastName` → `fullName`, `payment.method` via `valueOf` estoura `IllegalArgumentException` se desconhecido).
- **Aula 7.7:** `KafkaInvoiceIntegrationEventPublisher` valida (`ConstraintViolationException`) e espera ack por até 40s, convertendo falha em `EventPublishingException`.
- **~~O `InvoiceCanceled` ainda não sai~~:** resolvido na aula 8.3.
- **Comentários e logs copiados (achados de revisão):** o handler de `InvoiceIssued` no billing tinha o comentário "recebemos o evento de order placed para gerar invoice"; o `logReceived` do listener de fatura do ordering escrevia `partiton`; o `KafkaConfig` do billing mencionava "os tópicos próprios do ordering" e o prefixo `ordering.dlt.` (o código usa `billing.dlt.`).
- **Header `idempotency-key`:** os eventos de fatura não o carregam (o `IntegrationEvent` do billing devolve `null` por padrão; o contrato só exige o header nos eventos da outbox).

### Pendências

(Estado ao fim do módulo 7/8, antes da remoção da coreografia.)

- Billing publica direto no Kafka (after-commit), sem outbox: evento pode se perder após o commit. Defesa sugerida: job de conciliação (faturas `UNPAID` antigas ou `PAID` sem pedido pago) que republique.
- `processPayment` não checa o status antes do `capture()`: `GatewayTimeoutException`/`BadGatewayException` são retryable e podem cobrar de novo; duplicata do `InvoiceIssued` captura e só então o `assignPayment` lança `DomainException` ("cannot be edited") → DLT. Defesas: checar `isUnpaid()`/código de gateway antes; `invoiceId` como chave de idempotência no gateway; consultar pagamento antes de recapturar.
- Duplicata vira DLT nos dois lados: `DataIntegrityViolationException` (fatura dupla, billing) e `OrderStatusCannotBeChanged` (`InvoicePaid` duplicado, ordering). Correção sugerida: "já está no estado alvo" = sucesso idempotente, ou `runOnce`.
- Checklist aberto: nenhum evento perdido; duplicata como sucesso; cobrança idempotente; responder "onde parou o pedido X?" sem logs de três serviços.

### Testes

- **Aula 7.11 — Corrigindo testes:** não houve alteração em `src/test` nos diffs do billing nem do ordering; a seção ficou em aberto.
- Sugestão de experimento: derrubar os três brokers depois de o `OrderPlaced` ser consumido e antes do `send()` (breakpoint no `KafkaInvoiceIntegrationEventPublisher`) e observar o log e a tabela `invoice`.

## 10 — Saga coreografada

### Revisões e correções

- **Status no código (módulo 8):** C1 `InvoiceCanceled` → pedido cancelado (8.3); S1 `StockReservation` (8.4); S2 `StockService.withdrawAll` (8.5); S3 `OrderPaid` → reserva (8.6–8.9); S4 confirmação → `OrderReady` (8.10); C2 rejeição → estorno → `InvoiceCanceled` (8.11, com elo quebrado); P1 eventos de produto no Kafka (8.12).
- **Resolvidas durante o módulo:** `StockMovementEventListener.handle` não escutava nada, ganhou `@EventListener` (síncrono, dentro da tx) e o `ProductSoldOut` voltou a existir; `equals` do `Product` tinha virado comparação de referência, voltou a `callSuper = false` com comentário; comentário do `withdrawAll` desmentido pela refatoração foi corrigido; `StockServiceTest` não compilava e havia duas `InsufficientStockException` (o ajuste no Mongo passou a usar a de `domain/stock`).
- **Refatoração 8.5:** `StockService.restock(Product,int)` → `restock(UUID,int)`; `withdraw(Product,int)` → `withdraw(UUID,int,String orderId)`; novo `withdrawAll`; `QuantityInStockAdjustment.Result.isOutOfStock()/isRestocked()` → `StockMovementRegisteredEvent.causedSoldOut()/causedRestock()`; `StockMovement extends AbstractAggregateRoot` com `orderId`; `StockMovementEventListener` traduz.
- **8.7 — kit de consumidor do catálogo:** `KafkaConsumerTypeIdIdentifier` (tipo desconhecido → `ObjectNode` → handler default), `ErrorHandlingDeserializer` + `JacksonJsonDeserializer`, `KafkaConfig.defaultErrorHandler` 2s/4s/8s com DLT `product-catalog.dlt.ordering.order.events`, não-retryable (`DomainException`, `ConstraintViolationException`, `DataIntegrityViolationException`, `DuplicateKeyException`, `IllegalArgumentException`), grupo `product-catalog.order-events`.
- **8.9:** `StockReservationEventListener` publicava `StockReservationConfirmed/RejectedIntegrationEvent` (`reservationId`, `orderId`, `confirmedAt`/`rejectedAt`) com key `orderId` em `product-catalog.stock.events`; `KafkaConfig` com método privado `publish(...)` compartilhado por `ProductIntegrationEventPublisher` e `StockIntegrationEventPublisher`.
- **8.12:** `ProductSoldOut`/`Restocked`/`PlacedOnSale` em `product-catalog.product.events`; `StockMovementEventListener.handleEvictProductCache` faz `@CacheEvict` após qualquer movimento; `getIdempotencyKey()` virou `default null` no `IntegrationEvent` do catálogo; no ordering, a lógica de ajuste de carrinhos saiu de `ShoppingCartManagementApplicationService` para `ShoppingCartProductAdjustmentService`.
- **Mapper × assembler (8.3):** `OrderCanceled` usa `Mapper.convert` (ModelMapper) a partir do record `OrderCanceledEvent(OrderId, CustomerId, OffsetDateTime)` para destino com `String`/`UUID`.
- **`MANDATORY` removido (Fase 44):** o `@Transactional(propagation = MANDATORY)` do `QuantityInStockAdjustmentMongoDBImpl` (aula 8.5) saiu. Consequência para a corrida na reserva: sem o proxy `MANDATORY`, a exceção do `decrease()` não marca mais a transação como rollback-only; o `catch` do `reserveForOrder` grava a rejeição e o commit leva junto as baixas parciais que já aconteceram.
- **A corrida na reserva funcionava por acidente (antes da remoção do `MANDATORY`):** a `InsufficientStockException` nascida dentro do proxy `MANDATORY` marcava a tx como rollback-only; o `catch` gravava `REJECT`, o commit lançava `UnexpectedRollbackException` (nada gravado), a exceção não era não-retryable, o Kafka reentregava o `OrderPaid` e a pré-validação rejeitava do jeito certo. Sugestão: teste de dois pedidos disputando a última unidade.

### Pendências

(Estado ao fim do módulo 8, antes da remoção da coreografia.)

- **O estorno não publica `InvoiceCanceled`:** `refundByOrderId` chama `refund` e `updatePaymentStatus(REFUNDED)` sem `invoiceRepository.save(invoice)`; o `AbstractAggregateRoot` só publica no `save`. Cliente reembolsado, fatura `CANCELED`, pedido `PAID` para sempre. (No código atual o método continua sem `save`.)
- **Estorno é HTTP dentro da transação** (`refundByOrderId` `@Transactional`): prende conexão e repete o estorno se o commit falhar. Desenho seguro: carregar (tx) → estornar (sem tx) → gravar (tx).
- **Estorno sem idempotência:** `StockReservationRejected` duplicado chama `refund()` antes de qualquer checagem; só depois o `cancel()` recusa (`DomainException` → DLT).
- **Duplicata vira DLT no catálogo:** `DuplicateStockReservationException` é `DomainException` → `product-catalog.dlt.ordering.order.events`.
- **After-commit também no catálogo:** reserva gravada e evento perdido deixa o pedido `PAID` para sempre.
- **Contrato × código na key:** `messages.yml` declarava `KafkaStockReservationKey` (key = `reservationId`); o código publicava com `orderId`. (Desapareceu com a remoção dos eventos de reserva na Fase 44.)
- **`billing-scheduler`:** `CancelExpiredInvoicesApplicationServiceJDBCImpl` faz `UPDATE` direto, sem `Invoice.cancel()` nem `InvoiceCanceled`; o pedido da fatura vencida nunca é cancelado.
- **ModelMapper com `CustomerId`:** `OrderCanceled` e `OrderReady` por `Mapper.convert` a partir de records com `CustomerId`, sem converter para `UUID` no modo `STRICT`; falta teste unitário da conversão.
- **Detalhes menores:** `KafkaStockIntegrationEventListener` do ordering importava classes sem uso (`ProcessAcceptedCheckoutIntegrationCommand`, `InvoicePaidIntegrationEvent`, `KafkaConfig`, `UUID`); nos `Product*IntegrationEvent` do catálogo, `idempotencyKey` tem inicializador sem `@Builder.Default` (builder entrega `null`; hoje o caminho é o `Mapper`); `ProductPlacedOnSaleIntegrationEvent` publicado e ninguém consome.
- Checklist aberto: compensação do pivô publicando o próximo evento; chamadas externas fora da tx; compensações idempotentes; publicação com garantia; todo cancelamento passando pelo agregado; contrato igual ao código; saber o passo de cada saga sem logs.

### Testes

- Nada específico além dos itens de compilação de `StockServiceTest` (acima).

## 11 — Saga orquestrada: conceito e mapa

### Revisões e correções

- Nada removido além das referências a "doc vivo" e ao módulo 9 da trilha ("Saga e orquestração").

### Pendências

- Nenhuma.

### Testes

- Nenhum.

## 12 — Saga orquestrada: preparando o terreno

### Revisões e correções

**Removendo o checkout assíncrono — lista completa por camada:**

| Camada | Removido |
|---|---|
| Web | `MyOrderV2Controller`, `OrderAcceptedOutput`; `POST /api/v2/customers/me/orders` com `application/vnd.order-with-product.v2+json` e `application/vnd.order-with-shopping-cart.v2+json` |
| Gateway | `/api/v2/customers/**` do `Path` da rota do ordering no `gateway-ecommerce` |
| Aplicação | `CheckoutAsyncApplicationService`, `BuyNowAsyncApplicationService`, `ProcessAcceptedCheckoutApplicationService` |
| Portas de entrada | `ForBuyingWithShoppingCartAsync`, `ForBuyingProductAsync`, `ForProcessingCheckoutAccepted` |
| Comando | `ProcessAcceptedCheckoutIntegrationCommand` |
| Mensageria | `KafkaOrderIntegrationCommandListener`, `KafkaOrderIntegrationCommandPublisher`, `OutboxOrderIntegrationCommandPublisher`, porta `ForPublishingOrderIntegrationCommands` |
| Tópicos | Beans `NewTopic` de `ordering.order.commands` e `ordering.dlt.ordering.order.commands` |
| Configuração | `orderCommandTopicName` e `orderCommandsConsumerGroup` no `AlgaShopMessagingKafkaProperties` e nos YAMLs; alias `Ordering.ProcessAcceptedCheckoutIntegrationCommand` |
| Snapshot | `OrderSnapshot.shoppingCartId`; `OrderSnapshotAssembler.toSnapshot(Order, UUID)`, `toDomain` e conversores privados |

Também ficaram: `OrderPlacedIntegrationEventAssembler`, `BillingInputDisassembler`, `ShippingInputDisassembler`, `CheckoutInput`, `BuyNowInput`.

**Conferência do checkout assíncrono:**

- Todos os itens removidos; beans de tópico e DLT, propriedades e alias nos dois YAMLs removidos; itens preservados existem; nenhum teste Java referencia as classes removidas.
- `OrderSnapshotAssembler` não compilava: tinha recebido acessores `.value()` que não existem nos records do projeto (`Money(money)`, `Quantity(quantity)`, `Document(document)`…), `order.totalItems()` no lugar de `totalQuantity()` e um argumento a menos no `OrderItemSnapshot`. Voltaram os acessores originais; só a remoção pedida ficou.
- `KafkaStockIntegrationEventListener` não compilava: tirado o import de `ProcessAcceptedCheckoutIntegrationCommand` (e outros imports sem uso).
- Gateway sem `/api/v2/customers/**`; caminhos v1 e o `CircuitBreaker` ficaram.
- Comentários de `OrderSnapshot.java`, `OrderSnapshotAssembler.java` e `KafkaConfig.java` não falam mais do comando.
- Ambiente: tópicos antigos continuam no broker; conferir se a tabela `outbox` não tem linhas pendentes para `ordering.order.commands` antes de subir.

**Removendo a saga coreografada — detalhes:**

- ordering: `KafkaInvoiceIntegrationEventListener`, `KafkaStockIntegrationEventListener`; `InvoicePaid`, `InvoiceCanceled`, `StockReservationConfirmedIntegrationEvent`; `OrderManagementApplicationService` e porta `ForManagingOrders`; `OrderManagementApplicationServiceIT`; tópicos e grupos de fatura e estoque, dois DLTs, três aliases.
- billing: `KafkaOrderIntegrationEventListener`, `KafkaStockIntegrationEventListener`; `OrderPlacedIntegrationEvent`, seis snapshots de `application.order.snapshot`, `GenerateInvoiceInputAssembler`, `StockReservationRejectedIntegrationEvent`; tópicos e grupos de pedido e estoque, dois DLTs, dois aliases.
- catálogo: `KafkaOrderIntegrationEventListener`; `OrderPaidIntegrationEvent`, `OrderItemSnapshot`, `OrderPaidIntegrationEventAssembler` do catálogo; `StockReservationEventListener`, `StockIntegrationEventPublisher` (interface e bean), eventos de integração e de domínio da reserva; bean do tópico `product-catalog.stock.events`, DLT do tópico de pedidos, propriedades, três aliases.
- Ficou também `OrderPaidIntegrationEventAssembler` do ordering e `createDeadLetterTopic` no `KafkaConfig` do catálogo (sem uso, de propósito, para os tópicos de comando).
- `messages.yml`: saíram mensagens, headers e payloads de `StockReservationConfirmed/Rejected`, o trait `KafkaStockReservationKey` e o schema `StockReservationId` (a divergência de key da saga coreografada desaparece).

**Conferência da saga coreografada:** último alias sem vírgula nos dois YAMLs; imports órfãos no listener de fatura do billing limpos; `StockReservation` sem `AbstractAggregateRoot` (import sobrando tirado); nenhuma referência às classes e propriedades removidas em `src/` dos três serviços.

### Pendências

- Observar nas próximas aulas se o `KafkaInvoiceIntegrationEventListener` do billing (Listen to Yourself do `InvoiceIssued` → `processPayment`) continua ou se a cobrança também vira comando do orquestrador.
- Limpeza opcional do broker: `ordering.order.commands` e DLT; `product-catalog.stock.events`, `ordering.dlt.billing.invoice.events`, `ordering.dlt.product-catalog.stock.events`, `billing.dlt.ordering.order.events`, `billing.dlt.product-catalog.stock.events`, `product-catalog.dlt.ordering.order.events`.

### Testes

**Números:**

- Após remover o checkout assíncrono: `./gradlew test` do ordering com 304 testes, 0 falhas, nenhum teste Java alterado.
- Após remover a saga coreografada: unitários ordering 304, billing 48, catálogo 117 (0 falhas); contrato ordering 12, catálogo 17; ITs ordering 185 (já sem `OrderManagementApplicationServiceIT`), billing 20, catálogo 12 — depois das correções abaixo.

**Os testes que já estavam quebrados:** o primeiro `./gradlew check` depois da remoção falhou em 20 ITs do billing e 11 do catálogo, nenhum por causa da remoção (os ITs desses serviços não rodavam com sucesso desde o módulo 7):

| Serviço | Sintoma | Causa | Correção |
|---|---|---|---|
| billing | Contexto não sobe (20 ITs) | `application-test-env.yaml` sem Kafka: faltavam `algashop.messaging.kafka.*` (`@NotBlank`) e o `spring.json.type.mapping` lido por `@Value` no `KafkaConsumerTypeIdIdentifier` | Bloco `spring.kafka` no padrão do ordering (mapping, producer, consumer) + tópico e grupo de fatura; diferença: `spring.kafka.listener.auto-startup: false` para o listener do `InvoiceIssued` não entrar no grupo `billing.invoice-events` do cluster de desenvolvimento |
| billing | `shouldProcessInvoicePaymentAndCancelInvoice` | Verificava a chamada ao `InvoiceEventListener`, que virou `@TransactionalEventListener` (AFTER_COMMIT) na 8.3; o teste é `@Transactional` com rollback, o commit nunca acontece | Passou a verificar o `InvoiceCanceledEvent` publicado no save (`ApplicationEvents`) |
| billing | `PaymentGatewayServiceFastpayImplIT` passa sozinho, falha na suíte | Cada classe parava/subia o WireMock na porta 8788; o contexto Spring em cache guardava conexão com o servidor anterior → `NoHttpResponseException` | Um WireMock só por JVM: `startMock` não sobe se já estiver rodando, `stopMock` não para; reset de cenários no `@BeforeEach` mantém o isolamento |
| catálogo | `StockTransactionIT` não sobe | `KafkaConsumerTypeIdIdentifier` (8.7) exige o mapping, ausente no teste | Bloco `spring.kafka` no `application-test-env.yml`, padrão do ordering |
| catálogo | 7 ITs de `QuantityInStockAdjustment` | `@Transactional(propagation = MANDATORY)` (8.5) recusa a chamada direta dos testes de fatia e de concorrência | `MANDATORY` removido |

Por que tirar o `MANDATORY` em vez de abrir transação nos testes: o ajuste é um `findAndModify` condicional, atômico por documento; quando o chamador abre transação (`restock`, `withdraw`, `reserveForOrder`), o `MongoOperations` participa dela com ou sem a anotação. Envolver o teste de concorrência em transações Mongo mudaria o que ele testa (disputa com `WriteConflict` em vez da atomicidade do `findAndModify`). Perdeu-se a trava; ficou a atomicidade; testes Java do catálogo não mudaram.

**Roteiro de verificação — checkout assíncrono removido:**

1. Testes do ordering passando, sem alterar nenhum teste Java.
2. Checkout v1 com carrinho e compra direta v1, autenticado como o cliente `6e148bd5-47f6-4022-b9da-07cfaa294f7a`, Visa final 1022 (`creditCardId` `0199deda-9486-7d27-b40c-761f561a24d8`, pago na captura). Na época (coreografia ainda ativa), cada pedido percorria os seis eventos do módulo 8 e terminava em `Ordering.OrderReadyIntegrationEvent`, com `ready_at` preenchido.
3. `POST /api/v2/customers/me/orders` autenticado, pelo gateway e direto no ordering, com `Content-Type: application/vnd.order-with-shopping-cart.v2+json`: as duas respostas são `404 Not Found` (o media type certo garante que o 404 prova que o caminho saiu).
4. Nenhuma mensagem nova em `ordering.order.commands`; consumer group `ordering.order-commands` sem membros ativos.

**Roteiro de verificação — saga coreografada removida:**

1. Subir os três serviços sem erro (alias apontando para classe apagada derruba com `IllegalArgumentException`).
2. Checkout v1 com carrinho (mesmo cliente e cartão): o pedido para em `PLACED`.
   - `ordering.order.events` recebe só `Ordering.OrderPlacedIntegrationEvent`; nada novo em `billing.invoice.events` nem em `product-catalog.stock.events`;
   - `select id, status, placed_at, paid_at, ready_at from public."order" where id = '<orderId>';` (banco `ordering`): `PLACED`, `paid_at` e `ready_at` vazios;
   - `select id, status from public.invoice where order_id = '<orderId>';` (banco `billing`): nenhuma linha;
   - `db.stock_reservations.find({ orderId: "<orderId>" })` (mongosh, catálogo): nenhum documento.
3. Consumer groups: `ordering.invoice-events`, `ordering.stock-events`, `billing.order-events`, `billing.stock-events`, `product-catalog.order-events` sem membros ativos; `ordering.product-events` continua ativo.
4. Limpeza opcional do broker (lista em Pendências).

## 13 — Saga orquestrada: estado e persistência

### Revisões e correções

**Modelando a saga (`PlaceOrderSaga`)**
- Corrigido: `APPORVING_ORDER` → `APPROVING_ORDER`. O enum é gravado como texto na tabela da saga; renomear um valor depois de persistido exige migração de dados.
- Corrigido: import sem uso de `OrderStatus` em `SagaStatus`.
- Para ficar de olho: status e passo são validados separadamente. Os enums sozinhos não impedem combinações sem sentido, como `COMPENSATING` em `APPROVING_ORDER` ou `RUNNING` em `REFUNDING_INVOICE`. Quem garante a coerência são os métodos de transição (cada um mudando passo e status juntos, com `require` antes). Um teste unitário por transição é o lugar natural para travar isso.
- Na primeira versão do modelo as guardas privadas `require`, `changeStatusTo`, `changeStepTo` ainda não tinham chamador; os métodos de transição vieram nas etapas seguintes.
- Histórico de nome: o passo de estorno nasceu como `REFUNDING_INVOICE` (`CANCELLING_ORDER(INVOICING, REFUNDING_INVOICE)`) e foi renomeado para `CANCELLING_INVOICE` na etapa "Compensação: falta de estoque" (nome do comando que ele envia, mesmo padrão de `CANCELLING_ORDER`). Nenhuma saga chegou a gravar o nome antigo, então não houve migração de dados.
- Na V010 a coluna `failure` nasceu "reservada, ainda sem campo na entidade"; ganhou campo (`PlaceOrderSagaFailure`, na subclasse `PlaceOrderSagaPersistenceEntity`) na etapa "Compensação: pagamento recusado".

**Persistindo a saga**
- Corrigido: o campo se chamava `updateAt`, mas a coluna é `updated_at`. A estratégia de nomes do Spring Boot transformaria o campo em `update_at`; com `ddl-auto: none` nada acusaria na subida, e o primeiro `insert` falharia com "column update_at does not exist". Renomeado para `updatedAt`. Nenhum teste gravava saga ainda, por isso a suíte não pegou. Foi o `CheckoutApplicationServiceIT` (depois de "Onde a saga começa") que confirmou a correção no Postgres de teste.
- Corrigido: import sem uso (`LastModifiedBy`).
- Pendência registrada nesta etapa (resolvida depois em "O orquestrador recebe a resposta e avança"): o `add` monta uma entidade nova a cada chamada. Para atualizar uma saga já gravada: (1) a entidade nova chega com `createdAt` nulo, o `merge` copia o nulo por cima e viola o `not null`; (2) a `version` incrementada pelo banco não volta para o `PlaceOrderSaga`, e a gravação seguinte, com versão velha, daria `OptimisticLockException`. O `OrderPersistenceProvider` resolve os dois (`findById` decide entre `update` e `insert`, copia campos para a entidade carregada, `updateVersion` por reflexão).
  - Resolução: `createdAt` com `@Column(updatable = false)` (o `UPDATE` não inclui mais a coluna). `version` desatualizada "na prática não morde": cada resposta recarrega a saga do banco com a versão atual e o objeto não é reaproveitado depois do `add`. O lock otimista funciona: duas respostas concorrentes, e a segunda falha no `UPDATE ... where version = ?`.
- `PlaceOrderSagasPersistenceProvider` é `@Transactional(readOnly = true)` por padrão; só o `add` abre escrita.

**Onde a saga começa**
- O antigo `listen(OrderPlacedEvent)` do `OrderEventListener` virou `onOrderPlacedPublishIntegrationEvent` e ganhou o irmão `onOrderPlacedStartSaga`. O `CheckoutApplicationServiceIT` passou a exercitar a persistência da saga e foi ajustado para verificar os dois ouvintes.
- Estado intermediário desta etapa: a saga começava e parava; o coordenador ainda não enviava o `Issue Invoice`. Um pedido novo ficava `PLACED` com a saga `RUNNING` em `PLACING_ORDER`.

### Pendências
- Um IT do `PlaceOrderSagasPersistenceProvider` (gravar, ler, gravar de novo) é o que pegaria os problemas de `createdAt`/`version` no update.
- O índice parcial `idx_saga_instance_active` foi pensado para um job de recuperação/timeout de sagas ativas, que ainda não existe.

### Testes
- Persistindo a saga: `./gradlew check` do ordering verde (304 unitários, 12 de contrato, 185 de integração, com a V010 aplicada no Postgres de teste).
- Onde a saga começa: todo checkout v1 grava uma linha em `saga_instance`; `CheckoutApplicationServiceIT` cobre.

## 14 — Saga orquestrada: comandos e respostas

### Revisões e correções

**Entrada × saída (interfaces de integração)**
- `IntegrationEvent` servia para eventos publicados e consumidos; no consumidor, `getAggregateId()` existia só por simetria e nenhum código o chamava. Separado em `OutboundIntegrationEvent`, `InboundIntegrationEvent`, `OutboundIntegrationCommand` (com `getIdempotencyKey()` agora default `null`). Os eventos de produto do ordering perderam o `getAggregateId()`.
- Portas e adaptadores acompanharam nos três serviços (`ForPublishingOrderIntegrationEvents`, `InvoiceIntegrationEventPublisher`, `ProductIntegrationEventPublisher`, publishers Kafka e de outbox, `*PublishingException`). Billing e catálogo fizeram só o rename.
- Comentários corrigidos: as `OutboundIntegrationEvent` de billing e catálogo diziam "eventos de integração do order" (copiado do ordering); a do catálogo tinha perdido a explicação de key → partição → ordem por agregado; a do ordering ainda descrevia o lado consumidor. `InboundIntegrationEvent` ganhou comentário próprio.

**Contrato da orquestração (AsyncAPI)**
- ~~Corrigido: `CancelInvoiceCommand` passou a `Ordering.`~~ — revertido em "O billing executa o comando": a convenção para comandos é prefixo = serviço que executa (`Billing.CancelInvoiceIntegrationCommand`).
- Não há comando para `Approve order` (passo interno do ordering).
- O contrato ficou à frente do código: na época nenhum comando nem resposta existia em Java. Risco anotado: os `__TypeId__` do contrato têm de bater com os aliases do `spring.json.type.mapping` (lição da Fase 40).
- `$ref` dos quatro arquivos: nenhum quebrado.

**A outbox aprende a transportar comandos**
- V011 renomeia `event_type` → `message_type` (e `OutboxMessage.messageType`, `readMessageType` do recorder). Linhas pendentes continuam válidas. Como toda migração de schema, deve rodar com o serviço parado.
- V012 adiciona `correlation_id` e `reply_topic` à outbox.
- `OutboxRecorder` continua `@Transactional(propagation = MANDATORY)`.
- Corrigido: o comentário do `OutboxKafkaSender` ainda dizia `__TypeId__ = eventType` e não citava os headers novos. `OutboxDraft` ganhou comentário.
- `OutboxOrderIntegrationEventPublisher` foi o primeiro a usar o draft (sem correlação nem resposta).

**O primeiro comando: emitir a fatura**
- ~~Corrigido: alias `Billing.IssueInvoiceIntegrationCommand` trocado por `Ordering.`~~ — revertido: `Billing.` estava certo; ordering e billing usam `Billing.IssueInvoiceIntegrationCommand` e o contrato foi alinhado.
- Corrigido: `application-test-env.yaml` do ordering e do billing não tinham as properties novas (`@NotBlank`); o contexto dos ITs não subiria. Adicionadas; o ordering ganhou também o alias do comando.
- Comentários curtos em cada classe nova/alterada.
- Decisões: comando reaproveita `OrderSnapshotAssembler.toSnapshot(Order)` (que tinha ficado sem chamador desde a remoção do checkout assíncrono), itens ordenados por id para gerar sempre o mesmo JSON; `correlationId = sagaId`; quem consome o tópico o cria (billing cria `billing.invoice.commands`, ordering cria `ordering.saga.replies` e seu DLT).
- Properties: ordering `billing-invoice-commands-topic-name`, `saga-replies-topic-name` (depois `saga-replies-consumer-group`, `product-catalog-commands-topic-name`); billing `invoice-commands-topic-name`, `invoice-commands-consumer-group`.

**O billing executa o comando**
- Fixada a convenção dos `__TypeId__` (evento: quem publica; comando: quem executa; resposta: quem responde). As duas "correções" para `Ordering.` feitas antes foram revertidas. O YAML de teste do ordering, que tinha ficado com `Ordering.`, foi corrigido; o do billing ganhou o alias.
- Voltaram ao billing: os seis records de `application/order/snapshot/*` e o `GenerateInvoiceInputAssembler` (agora a partir do comando), que tinham saído na remoção da coreografia.
- `KafkaConfig` do billing ganhou o DLT `billing.dlt.billing.invoice.commands`.
- Comentários no listener, no `InboundIntegrationCommand` e no assembler.

**Resposta assíncrona: guardando o reply pendente**
- Corrigido: o comentário do listener dizia "inicia transactional paralelo"; é o contrário (uma transação só, compartilhada pelas duas escritas).
- `TransactionTemplate` no listener (e não `@Transactional` no método) porque o método também valida headers e monta o input; o template delimita exatamente o trecho que escreve.
- V0004 cria `pending_reply` (`invoice_id` PK+FK, `order_id`, `correlation_id`, `reply_topic`, `created_at`). O comentário da migração a chama de "como se fosse um outbox".

**Enviando o reply ao coordenador**
- Corrigido: o comentário de `OutboundIntegrationReply` falava em "eventos de uma mesma fatura", mas a key é o `orderId` e o tipo é resposta. Classes novas ganharam comentários sobre envio manual e `REQUIRES_NEW`. YAML de teste do billing ganhou os aliases das respostas.
- `KafkaInvoiceIntegrationReplyPublisher` valida a resposta, monta o `ProducerRecord`, põe `kafka_correlationId`, espera o ack (`send().get(40s)`); `ReplyPublishingException` para falha.
- Fatura gerada pela API REST (sem saga) não tem pendente: o `find` volta vazio e nada é respondido.

**O orquestrador recebe a resposta e avança**
- Peças: `KafkaPlaceOrderSagaReplyListener`, `InboundIntegrationReply`, `InvoicePaidIntegrationReply`, `onInvoicePaid`, `moveToReservingStock`, `ReserveStockIntegrationCommand` (alias `Stock.ReserveStockIntegrationCommand`), nova rota no `PlaceOrderSagaCommandTopics`, `PlaceOrderSagaNotFoundException`, catálogo `stock-commands-topic-name` + bean do tópico.
- Resolvida a pendência da persistência (ver doc 13).

**Como está o fluxo da saga (revisão)**
- Corrigido: YAMLs de teste do ordering (sem `saga-replies-consumer-group`, `product-catalog-commands-topic-name` e os dois aliases) e do catálogo (sem `stock-commands-topic-name`, que o `StockTransactionIT` exige); os ITs não subiriam.
- Corrigido: contrato — `ReserveStock` passou a `Stock.ReserveStockIntegrationCommand`, como no código.
- Corrigido: comentário "ignora o command" (é uma resposta). Classes novas ganharam comentários.

**O catálogo reserva e responde (`@SendTo`)**
- Corrigido: config de teste do catálogo sem `stock-commands-consumer-group` (`@NotBlank`) e sem os aliases; `StockTransactionIT` não subiria.
- Corrigido: o contrato declarava `ProductCatalog.StockReservation*IntegrationReply`; alinhado ao código: `Stock.*`.
- Corrigido: DLT do tópico de comandos não declarado. O recoverer publica na mesma partição do record original, então o DLT precisa existir com as mesmas 3 partições; sem o `NewTopic`, nasceria por auto-criação sem garantia. Declarado com o `createDeadLetterTopic` (deixado de propósito na remoção da coreografia).
- Comentários no listener (`@SendTo`), no query service e no `OutboundIntegrationReply`.
- Peças: `StockReservationQueryService` + `StockReservationOutput` + `StockReservationQueryServiceImpl`; `StockReservationNotFoundException` (`DomainEntityNotFoundException`, não-retryable); `KafkaConfig.stockCommandsDlt`; `reserveForOrder` passou a devolver o id da reserva.

**Caminho feliz completo**
- Corrigido: faltava `@Transactional` no `onStockReservationConfirmed` (o `onInvoicePaid` tinha). Funcionava porque `orders.add` e `sagas.add` abrem cada um a própria transação, mas pedido `READY` e saga `SUCCEEDED` eram gravados separados. Se a saga falhasse depois (lock otimista), a reentrega acharia a saga em `RESERVING_STOCK` e chamaria `markAsReady()` num pedido já `READY` → `OrderStatusCannotBeChanged` (não-retryable), DLT, pedido pronto e saga travada para sempre. Com a anotação, commitam juntos.
- Corrigido: YAML de teste do ordering ganhou o alias da resposta nova; métodos novos comentados.
- Dados de teste: `afterMigrate.sql` passou a limpar `saga_instance` (ordering) e `pending_reply` (billing; redundante com o `TRUNCATE invoice CASCADE`, mas explícito).
- `endSucceeded()` só aceita `RUNNING` em `APPROVING_ORDER` (`require(canEndSucceeded())`).

### Pendências
- Comando não passa por Bean Validation antes de ir para a outbox (o `OutboxRecorder` não valida); as anotações do `IssueInvoiceIntegrationCommand` só protegem o lado consumidor.
- Envio da outbox com headers novos sem teste: um IT do `OutboxKafkaSender` com correlação e resposta preenchidas travaria esse caminho.
- Duplicata de `IssueInvoice` vira DLT: a reentrega bate no `UNIQUE (order_id)`, lança `DataIntegrityViolationException` (não-retryable) **antes** de registrar o pendente. Se a primeira resposta se perdeu, a reentrega não a recupera; o caso "fatura já existe para este pedido" deveria reenviar o reply.
- `pending_reply.created_at` sem uso; pendentes antigos são, por definição, respostas que não saíram (base para um job de reenvio).
- Ordem "enviar, depois apagar": envio falha → pendente fica (job pode reenviar); envio funciona e `delete` falha → pendente fica e um reenvio geraria duplicata. At-least-once.
- Ponto frágil: no `InvoiceEventListener`, o evento de integração é publicado **antes** do reply. Se o Kafka falhar no evento, a exceção é engolida (`AFTER_COMMIT`) e o reply nem é tentado; a saga para em `INVOICING` até alguém reenviar. Sugestão: separar o reply num `@TransactionalEventListener` próprio ou enviá-lo antes do evento.
- Limpeza sugerida: `KafkaInvoiceIntegrationReplyPublisher` tem um bloco comentado com código antigo do publisher de eventos.
- Guarda de idempotência estreita: `onInvoicePaid` usa `if (saga.isReservingStock() || saga.hasSucceeded())`; `onStockReservationConfirmed` usa `hasSucceeded()`. Se a saga já foi além (compensando, terminada), a resposta repetida passa pela guarda, `moveToReservingStock()` lança `SagaTransitionException` (não-retryable) e vai para o DLT (não corrompe, só polui). Forma robusta: `if (!saga.isRunningAt(INVOICING)) ignore`, antes de carregar o pedido (hoje o pedido é buscado mesmo quando a resposta vai ser ignorada).
- `PlaceOrderSagaNotFoundException` estende `RuntimeException` → retryable: três tentativas com backoff antes do DLT, para uma saga que não vai passar a existir.
- `@SendTo` do catálogo — reply pode se perder (dual-write): a reserva commita no Mongo e o reply sai depois. Se o envio falhar, o Kafka reentrega, o `reserveForOrder` esbarra no `DuplicateStockReservationException` (não-retryable), DLT sem resposta, saga parada em `RESERVING_STOCK`. Correção: se a reserva do pedido já existe, reler e responder de novo.
- `beanValidationUtil.validate(reservationOutput)` no catálogo valida o output, que não tem constraints; quem deveria ser validado é o reply.
- Key do reply com `@SendTo`: quem monta o record é o framework e ninguém chama `getAggregateId()`. O comentário de `OutboundIntegrationReply` promete key = `orderId`; conferir na GUI do Kafka qual key a resposta realmente leva.
- Pendências históricas já fechadas nas etapas seguintes: billing não consumia o comando; resposta ao orquestrador não existia; ordering não consumia `ordering.saga.replies`; `InvoiceCanceledReply` e `StockReservation*Reply` sem handler (caíam no default e iam ao DLT do ordering).
- Observação da remoção da coreografia que se confirmou: o `KafkaInvoiceIntegrationEventListener` do billing (Listen to Yourself do `InvoiceIssued`, aula 7.8) continua sendo quem dispara a cobrança que produz o `InvoicePaid`/`InvoiceCanceled` usado pelo reply.

### Testes
- Interfaces de entrada × saída: ordering `check` verde (304 unitários, 12 de contrato, 185 de integração); billing e catálogo compilam e os unitários passam.
- Outbox com comandos: ordering `check` verde (304, 12, 185). `CheckoutApplicationServiceIT` grava o `OrderPlaced` na outbox, exercitando as colunas da V011/V012.
- Primeiro comando: ordering `check` verde (304, 12, 185); billing `check` verde (48 unitários, 20 de integração). `CheckoutApplicationServiceIT` executa o caminho inteiro (pedido, saga em `INVOICING`, comando na outbox).
- Billing executa o comando: billing `check` verde (48, 20); ordering verde (304, 12, 185). Nos testes do billing o listener não sobe (`auto-startup: false`), então o consumo do comando não tem teste próprio; um IT que envie o comando e verifique a fatura fecharia o buraco.
- Reply pendente: billing `check` verde (48, 20, com a V0004). O registro do pendente não tem teste próprio (listener não sobe nos testes).
- Envio do reply: billing `check` verde (48, 20). Sem teste: os ITs rodam em transação com rollback e o `AFTER_COMMIT` nunca dispara.
- Orquestrador avança: ordering `check` verde (304, 12, 185); catálogo `check` verde (117, 17, 12). `onInvoicePaid` sem teste; sugeridos um IT (saga em `INVOICING` → pedido `PAID`, saga `RESERVING_STOCK`, comando na outbox) e outro chamando duas vezes (idempotência).
- Catálogo `@SendTo`: catálogo `check` verde (117 unitários, 17 de contrato, 12 de integração). Listener sem teste próprio.
- Caminho feliz completo: ordering `check` verde (304, 12, 185); billing verde (48, 20). Transições do coordenador sem teste próprio.

## 15 — Saga orquestrada: compensações

### Revisões e correções

**Compensação: pagamento recusado — o bug da saga parada em `RUNNING` / `INVOICING`**
- Sintoma: depois de processar o `InvoiceCanceledReply`, o pedido aparecia `CANCELED` e a saga continuava `RUNNING` / `INVOICING`. Dois defeitos somados:
  1. Condição de `canCompensateOrder()` invertida: era `status == COMPENSATING && step.canNotChangeTo(CANCELLING_ORDER)`; virou `step.canChangeTo(CANCELLING_ORDER)`. Como `CANCELLING_ORDER(INVOICING, ...)` declara que pode vir de `INVOICING`, `canNotChangeTo` era falso em `INVOICING`, o `require` falhava e `compensateOrder()` lançava `SagaTransitionException`.
  2. `onInvoiceCanceled` sem `@Transactional`. Sequência: (1) `failWithPaymentRefused()` só em memória; (2) `order.markAsCanceled()` + `orders.add(order)` commitou sozinho (o `add` do provider abre a própria transação); (3) `compensateOrder()` lançou `SagaTransitionException`; (4) `sagas.add(saga)` nunca rodou. A mensagem foi direto para o DLT (`SagaTransitionException` é não-retryable).
- Correções: condição invertida; `@Transactional` no `onInvoiceCanceled`; compensação idempotente `if (!order.isCanceled()) order.markAsCanceled()` (sem isso, reprocessar esbarraria em `OrderStatusCannotBeChanged` e as sagas presas não poderiam ser recuperadas pelo DLT); typo `canEndCompesated` → `canEndCompensated`; alias da resposta no YAML de teste; teste unitário novo `PlaceOrderSagaTest` (caminho feliz, compensação por pagamento recusado, recusa de compensar saga ainda `RUNNING`). Com a condição antiga, `shouldCompensateWhenPaymentIsRefused` falha; com a correção, passa. Era a primeira transição sem nenhum teste, e o defeito estava nela.
- Lição: cada método do coordenador é uma unidade e precisa de uma transação que a abrace (a mesma do `onStockReservationConfirmed`, agora com consequência visível).
- Recuperação das sagas presas (pedidos `CANCELED`, saga `RUNNING`/`INVOICING`, resposta no DLT `ordering.dlt.ordering.saga.replies`):
  - Preferível: republicar a mensagem do DLT em `ordering.saga.replies`; com a correção, a saga compensa, o pedido já cancelado é preservado e ela termina `COMPENSATED`.
  - Só em desenvolvimento, corrigir no banco:
    ```sql
    update public.saga_instance
       set status = 'COMPENSATED', step = 'CANCELLING_ORDER', failure = 'PAYMENT_REFUSED',
           version = version + 1, updated_at = now()
     where status = 'RUNNING' and step = 'INVOICING'
       and aggregate_id in (select id::text from public."order" where status = 'CANCELED');
    ```
- Peças desta etapa: `InvoiceCanceledIntegrationReply` + handler (alias `Billing.InvoiceCanceledIntegrationReply`); `PlaceOrderSagaFailure` gravado na coluna `failure` reservada desde a V010; `failWithPaymentRefused`, `compensateOrder`, `endCompensated`, `isCompensated`, `hasFailed`.

**Compensação: falta de estoque**
- Renomeado `REFUNDING_INVOICE` → `CANCELLING_INVOICE` (sem migração; nenhuma saga gravou o nome antigo).
- Peças: `StockReservationRejectedIntegrationReply` + handler (alias `Stock.StockReservationRejectedIntegrationReply`), `failWithOutOfStock`, `compensateInvoice`, `wasStockRejected` (guarda contra repetida), `CancelInvoiceIntegrationCommand` (alias `Billing.CancelInvoiceIntegrationCommand`, mesmo tópico do `IssueInvoice`), `onStockReservationRejected`.
- `@Transactional` movido para a classe `PlaceOrderSagaCoordinator`: duas vezes um método novo nasceu sem a anotação.
- Comentários nos métodos novos e no `@Transactional` da classe.
- Estado intermediário desta etapa: o billing ainda não executava o `CancelInvoice` (só tinha handler de `IssueInvoice`; o default lança). O comando ia para o DLT do billing, a saga ficava `COMPENSATING`/`CANCELLING_INVOICE` e o pedido `PAID`, com o cliente cobrado. O plano listava: handler + alias; chamar `refundByOrderId` (corrigindo o que o módulo 8 já apontava: sem `save`, `InvoiceCanceledEvent` nunca publicado, HTTP do estorno dentro da transação); registrar um novo `pending_reply` para o `CancelInvoice`; idempotência do estorno. O item do `pending_reply` deixou de ser necessário com o `@SendTo` (etapa seguinte).
- Detalhe: `onStockReservationRejected` chama `findOrder(...)` e descarta o resultado (só confirma que o pedido existe, ao custo de uma consulta). Ainda está assim no código.

**O billing estorna e responde (`@SendTo`)**
- Corrigido: YAML de teste do billing ganhou o alias `Billing.CancelInvoiceIntegrationCommand`. Handler ganhou comentário com o `@SendTo` e o contraste com o `IssueInvoice` (no código, o comentário aparece duplicado, com duas redações do mesmo texto, acima do handler do `CancelInvoice`).

### Pendências
- Guardas de resposta repetida seguem crescendo por lista (`isCompensated()`, `wasStockRejected()`); a forma por "passo esperado" é a mais robusta.
- Reentrega do `CancelInvoice` estorna de novo (dual-write de todo `@SendTo`): a fatura cancelada commita e a resposta sai depois. Se o envio falhar ou o comando chegar duplicado, o handler chama `refund` no gateway outra vez, e só depois o `cancel()` lança "already canceled" (`DomainException`, não-retryable): DLT, sem resposta, cliente possivelmente estornado duas vezes. Guarda: se a fatura já está `CANCELED`, pular o estorno e só montar a resposta.
- `refundByOrderId` continua sem `save`: a fatura é gravada por dirty checking, mas o `InvoiceCanceledEvent` não é publicado e `billing.invoice.events` não recebe o `InvoiceCanceled` do estorno. A saga não depende disso (resposta via `@SendTo`), mas outros assinantes ficariam sem o evento.
- Chamada HTTP do estorno continua dentro da transação do `refundByOrderId`.

### Testes
- Pagamento recusado: ordering `check` verde, com o `PlaceOrderSagaTest`.
- Verificação manual (pagamento recusado): checkout com um cartão que o FastPay recusa → `select status, step, failure from saga_instance where aggregate_id = '<pedido>'` deve mostrar `COMPENSATED`, `CANCELLING_ORDER`, `PAYMENT_REFUSED`, com o pedido `CANCELED`.
- Falta de estoque: ordering `check` verde. `PlaceOrderSagaTest` ganhou `shouldCompensateWhenStockIsRejected` (caminho inteiro até `COMPENSATED` com `OUT_OF_STOCK`) e `shouldNotCancelInvoiceWhenPaymentWasRefused` (sem pagamento não há fatura a estornar; transição para `CANCELLING_INVOICE` recusada). 5 testes no total.
- Billing estorna: billing `check` verde (48 unitários, 20 de integração). Handler do `CancelInvoice` sem teste próprio.

## 16 — Saga orquestrada: robustez

### Revisões e correções

**Validar antes de começar: quantidade na compra**
- NPE no mapeamento: o adaptador fazia `new Quantity(productResponse.getQuantityInStock())`, e `Quantity` não aceita nulo. Bastava o campo faltar (respostas antigas no Redis antes da mudança, stub WireMock dos testes) para a compra quebrar com `NullPointerException`. O domínio já tratava `quantityInStock == null` como "quantidade desconhecida, valide só a disponibilidade"; o adaptador passou a fazer o mesmo.
- Stub WireMock `get-product-by-id-v1.json` ganhou `"quantityInStock": 100`.
- `ProductCatalogServiceIT` não compilava (construtor antigo do `ProductResponse`, 4 argumentos); passou a informar a quantidade.
- O catálogo já expunha `quantityInStock` no `ProductDetailOutput`; o `ProductResponse` do ordering passou a ler o campo.

**Comandos da saga sem outbox**
- Antes: o ordering não subia com `algashop.messaging.outbox.enabled=false` (a porta `ForPublishingPlaceOrderSagaIntegrationCommands` só tinha a implementação de outbox). Criado `KafkaPlaceOrderSagaIntegrationCommandPublisher`.
- Revisão: o publicador atende o enunciado (ativação, tópico pela mesma decisão do outbox, headers, valor, envio bloqueante de 40 s, `CommandPublishingException`, nenhuma transação própria). Ganhou comentário curto.
- Observação: o `catch` pega a `KafkaException` do Spring, mas não a `org.apache.kafka.common.KafkaException`, que o `send` pode lançar de forma síncrona (falha de serialização, por exemplo). Ela escaparia sem virar `CommandPublishingException` (a transação ainda volta, com outra exceção). Mesmo padrão do `KafkaOrderIntegrationEventPublisher`; o publisher do catálogo trata as duas.
- `idempotency-key`: na outbox sempre (id da linha); no Kafka direto só se o comando tiver chave, e hoje nenhum tem.

**Validando os payloads no consumidor**
- `KafkaBeanValidationConfigurer` criado no billing e no catálogo (`infrastructure/kafka`). O ordering já registrava desde a resiliência do consumer.
- Revisão: os dois configurers atendem ao enunciado. O comentário deles tinha uma frase quebrada e passou a citar também o destino do payload inválido (DLT, sem retry).
- `KafkaConfig` dos dois serviços não mudou: `MethodArgumentNotValidException` já está na lista de fatais padrão do `DefaultErrorHandler` (sem as esperas de 2, 4 e 8 s).
- Comentário desatualizado no ordering: o `KafkaBeanValidationConfigurer` do ordering ainda diz que o inválido sofre "10 tentativas sem backoff" e que não há DLQ; isso deixou de valer desde a resiliência do consumer (DLT por tópico e exceções não-retryable).

### Pendências
- O carrinho valida só o que está sendo adicionado: item com 3 unidades, estoque 4, cliente adiciona mais 3 → a checagem compara 3 com 4, passa, e o item fica com 6. Deveria comparar com o total do item depois da soma.
- O checkout do carrinho não valida quantidade: o `CheckoutService` monta o produto a partir do item do carrinho com `quantityInStock = null`, então só a disponibilidade é conferida. Um carrinho montado ontem passa direto para a saga, e quem barra é a compensação.
- `Product.isOutOfStock()` (privado) ficou sem uso.
- Comentário do `KafkaBeanValidationConfigurer` do ordering desatualizado (acima).
- `catch` do `KafkaPlaceOrderSagaIntegrationCommandPublisher` não cobre a `KafkaException` do cliente Apache (acima).
- "Próximas aulas (a documentar)": a comparação com a coreografia vai seguir as perguntas do balanço da 8.13 — onde fica o estado "em que passo está o pedido 42"? Quem decide a compensação? O que acontece com os três kits de consumidor copiados?

### Testes
- Quantidade na compra: ordering `check` verde (309 unitários, 12 de contrato, 185 de integração).
- Sem outbox: `KafkaPlaceOrderSagaIntegrationCommandPublisherTest` (sem Spring, Kafka ou Postgres; `KafkaTemplate` mockado, `PlaceOrderSagaCommandTopics` real, propriedades à mão):
  - `shouldSendIssueInvoiceToBillingWithKeyAndCorrelationHeaders`: tópico `billing.invoice.commands`, key = orderId, valor é o próprio comando, `kafka_correlationId` e `kafka_replyTopic` corretos, sem `idempotency-key`;
  - `shouldSendCancelInvoiceToBilling`: `CancelInvoice` também vai para `billing.invoice.commands`;
  - `shouldSendReserveStockToProductCatalog`: `ReserveStock` vai para `product-catalog.stock.commands`;
  - `shouldThrowCommandPublishingExceptionWhenKafkaDoesNotConfirm`: `failedFuture` vira `CommandPublishingException`;
  - `shouldRejectUnknownCommandWithoutSendingAnything`: comando sem rota → `IllegalArgumentException` e nenhuma interação com o `KafkaTemplate`.
  - Ordering `check` verde: 314 unitários, 12 de contrato, 185 de integração.
- Roteiro manual sem outbox:
  1. `algashop.messaging.outbox.enabled: false` no `application-development-env.yaml` do ordering; o serviço tem de subir.
  2. Checkout v1 com o Visa final 1022 e estoque suficiente: no log, o `IssueInvoiceIntegrationCommand` é publicado em `billing.invoice.commands` antes de `Place order saga started`. Pedido `READY`, saga `SUCCEEDED` / `APPROVING_ORDER`.
  3. GUI do Kafka, mensagens de `billing.invoice.commands` e `product-catalog.stock.commands`: key, `__TypeId__` (`Billing.IssueInvoiceIntegrationCommand` / `Stock.ReserveStockIntegrationCommand`), `kafka_correlationId` igual ao `saga_id` da `saga_instance`, `kafka_replyTopic`.
  4. `select count(*) from outbox where aggregate_id = '<orderId>'` deve devolver zero.
  5. Visa final 1030: saga `COMPENSATED` / `CANCELLING_ORDER` / `PAYMENT_REFUSED`.
  6. Produto com saldo menor que a compra: `Billing.CancelInvoiceIntegrationCommand` em `billing.invoice.commands`, saga `COMPENSATED` / `CANCELLING_ORDER` / `OUT_OF_STOCK`. (Com a validação de quantidade, só chega à saga se o estoque mudar entre compra e reserva, ou pelo checkout do carrinho.)
  7. Opcional: parar os containers do Kafka e fazer um checkout; a requisição demora e falha, sem pedido nem saga gravados.
  8. Voltar `outbox.enabled` para `true`.
- Validação no consumidor: `KafkaBeanValidationConfigurerTest` em billing e catálogo (sem Spring, Kafka ou banco; `LocalValidatorFactoryBean` à mão com `afterPropertiesSet()`, `new KafkaListenerEndpointRegistrar()`, violações num `BeanPropertyBindingResult`): registrar devolve o mesmo Validator; payload inválido aponta o campo (billing `CancelInvoiceIntegrationCommand("")` → `orderId`; catálogo `ReserveStock` sem itens → `items`); payload válido não aponta nada.
  - Billing `check` verde (51 unitários, 20 de integração); catálogo `check` verde (120 unitários, 17 de contrato, 12 de integração). ITs existentes continuam passando com a validação ligada.
- Roteiro manual da validação (GUI do Kafka, os três serviços no ar):
  1. `billing.invoice.commands`, chave `01K00000000000000000000000`, `__TypeId__` `Billing.CancelInvoiceIntegrationCommand`, `kafka_correlationId` UUID qualquer, `kafka_replyTopic` `ordering.saga.replies`, valor `{ "orderId": "" }` → chega na hora a `billing.dlt.billing.invoice.commands`; o log do billing não tem `Received CancelInvoiceIntegrationCommand`.
  2. `billing.invoice.events`, `__TypeId__` `Billing.InvoiceIssuedIntegrationEvent`, valor sem `invoiceId` → `billing.dlt.billing.invoice.events`.
  3. `product-catalog.stock.commands`, `__TypeId__` `Stock.ReserveStockIntegrationCommand`, mesmos headers de correlação, valor `{ "orderId": "01K00000000000000000000000", "items": [] }` → `product-catalog.dlt.product-catalog.stock.commands`; nenhum reply novo em `ordering.saga.replies` e nenhuma reserva para esse pedido.
  4. Nas três mensagens da DLT, o header `kafka_dlt-exception-message` aponta o campo violado.
  5. Checkout v1 com Visa final 1022 e estoque suficiente: pedido `READY`, saga `SUCCEEDED` / `APPROVING_ORDER`. Produto com saldo menor: `COMPENSATED` / `CANCELLING_ORDER` / `OUT_OF_STOCK` (único caminho que envia o `CancelInvoice`; a validação de quantidade barra esse caso na compra direta).
  6. Durante os checkouts, nenhuma mensagem nova nas DLTs além das três produzidas à mão.

