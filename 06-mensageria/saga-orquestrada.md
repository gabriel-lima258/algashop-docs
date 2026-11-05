# Saga orquestrada: um lugar que sabe em que passo cada pedido está

> A [saga coreografada](./saga-coreografada.md) terminou com um balanço honesto: funciona, mas o fluxo só existe no mapa, e quando um evento se perde a saga não falha, ela só para. A orquestração troca isso por um **coordenador** que conhece os passos, guarda em que ponto cada pedido está e manda **comandos** a cada serviço. Este doc acompanha o módulo 9 da trilha ("Saga e orquestração").
> Doc vivo: começa pela preparação do terreno, antes de qualquer orquestrador existir. Duas demolições primeiro: o checkout assíncrono e a saga coreografada.
> Código real (desta etapa): `algashop-ordering/.../infrastructure/adapters/in/web/order/MyOrderController.java` (único caminho de criação de pedido), `core/application/checkout/CheckoutApplicationService.java` e `BuyNowApplicationService.java`, `core/application/order/snapshot/OrderSnapshot*.java`, `infrastructure/config/kafka/`; `api-gateway-ecommerce/src/main/resources/application-base.yml`.

---

## O mapa da orquestração

![SEC & Ordering Workflow](../images/Workflow-orquestra-resumo.png)

O resumo mostra a mudança de forma em uma imagem: em vez de uma corrente em que cada serviço puxa o próximo, há **um coordenador** (o ordering, como *SEC*, o coordenador de execução da saga) que manda três comandos (`Issue Invoice`, `Reserve Stock`, `Approve order`) e recebe uma **resposta** para cada um: sucesso (verde) ou falha (vermelho).

![Commands & Reply - Ordering Workflow](../images/Workflow-orquestra-detalhado.png)

O detalhado é a tabela que guia o módulo. Cada linha é um passo da saga, e as colunas separam quatro coisas que na coreografia viviam misturadas num evento só:

| Coluna | O que é | Exemplo |
|---|---|---|
| **Step** / **Compensating** | O passo e o que o desfaz | `Invoicing` / `Cancel invoice` |
| **Command** | O pedido do orquestrador a um participante | `Issue Invoice` |
| **Reply** | A resposta endereçada ao orquestrador | `Invoice Paid` ou `Invoice Canceled` |
| **Success / Failure Event** | O fato que o serviço anuncia para quem quiser ouvir | `Order Paid`, `Invoice Canceled` |

Duas leituras para guardar desde já:

- **Reply ≠ evento.** A resposta tem um destinatário (o orquestrador) e faz a saga andar. O evento continua sendo um anúncio sem destinatário. Na coreografia, o mesmo `InvoicePaid` fazia os dois papéis.
- **O pivô mudou de lugar.** Na [saga coreografada](./saga-coreografada.md#os-três-tipos-de-passo) o pivô era a cobrança; neste mapa o pivô marcado é **Reserving Stock**. Os passos antes dele têm compensação (`Cancel order`, `Cancel invoice`); o passo depois (`Approving order`) não tem, só precisa dar certo.

### O fluxo, passo a passo

Lendo o mapa detalhado linha por linha, com o ordering no papel de orquestrador:

| # | Passo (Step) | Comando enviado | Respostas possíveis | Evento anunciado | Compensação |
|---|---|---|---|---|---|
| 1 | **Placed Order** | — (a saga começa aqui, na transação do checkout) | — | `Order Placed` | `Cancel order` → `Order Canceled` |
| 2 | **Invoicing** | `Issue Invoice` → billing | `Invoice Paid` ou `Invoice Canceled` | `Invoice Paid`, `Order Paid` / `Invoice Canceled` | `Cancel invoice` |
| 3 | **Reserving Stock** (pivô) | `Reserve Stock` → catálogo | `Reservation Confirmed` ou `Reservation Rejected` | `Stock Reservation Confirmed` / `Stock Reservation Rejected` | — (é o ponto de não retorno) |
| 4 | **Approving order** | `Approve order` → ordering | — | `Order Ready` | — (só precisa dar certo) |

Repare que o passo 1 não tem comando: ele **é** o checkout v1. É por isso que o checkout assíncrono precisou sair (próxima seção): a saga nasce na mesma transação que grava o pedido.

### Caminho feliz

```mermaid
sequenceDiagram
    participant O as ordering (orquestrador)
    participant B as billing
    participant C as product-catalog
    Note over O: 1. Placed Order<br/>(checkout v1, mesma tx: Order + início da saga)
    O->>B: Issue Invoice (comando)
    B-->>O: Invoice Paid (resposta)
    Note over O: markAsPaid → Order Paid
    O->>C: Reserve Stock (comando)
    C-->>O: Reservation Confirmed (resposta)
    Note over O: 4. Approve order → markAsReady → Order Ready
```

### Caminhos de falha

```mermaid
flowchart TD
    A["Issue Invoice"] -->|"Invoice Canceled<br/>(pagamento recusado)"| X1["Cancel order<br/>→ Order Canceled"]
    A -->|"Invoice Paid"| R["Reserve Stock (pivô)"]
    R -->|"Reservation Rejected<br/>(sem estoque)"| X2["Cancel invoice<br/>(estorno)"]
    X2 --> X3["Cancel order<br/>→ Order Canceled"]
    R -->|"Reservation Confirmed"| OK["Approve order<br/>→ Order Ready"]
```

- **Pagamento recusado** (resposta `Invoice Canceled`): só o passo 1 precisa ser desfeito. O orquestrador cancela o pedido.
- **Estoque rejeitado** (resposta `Reservation Rejected`): a saga anda para trás desfazendo os passos já commitados, **na ordem inversa**: primeiro `Cancel invoice` (o estorno, que usa o `refundByOrderId` preservado no billing), depois `Cancel order`.

A diferença para a coreografia está em **quem encadeia as compensações**. Na [saga coreografada](./saga-coreografada.md#811--estornando-o-pagamento-como-compensação), a rejeição do estoque virava estorno no billing, o estorno virava `InvoiceCanceled`, e o ordering reagia cancelando o pedido: três serviços, cada um sabendo só o seu pedaço. Aqui o orquestrador **recebe** a rejeição e **decide** as duas compensações. A cadeia passa a estar escrita em um lugar só.

### Coreografia × orquestração, no mesmo fluxo

| Pergunta | Coreografia (módulo 8, removida) | Orquestração (módulo 9) |
|---|---|---|
| Quem decide o próximo passo? | Cada serviço, ao reagir ao evento anterior | O ordering, ao receber a resposta |
| Como o billing sabe que deve emitir a fatura? | Assina `OrderPlaced` | Recebe o comando `Issue Invoice` |
| Para quem vai o resultado? | Para quem quiser ouvir (evento) | Para o orquestrador (reply) **e**, se fizer sentido, um evento para o resto |
| Onde fica "em que passo está o pedido 42"? | Em lugar nenhum; é preciso correlacionar três serviços | No estado da saga, no ordering |
| Quem desfaz o quê na falha? | Emerge da cadeia de eventos | O orquestrador manda as compensações |
| Qual é o pivô? | A cobrança | A reserva de estoque |

---

## Antes de orquestrar: removendo o checkout assíncrono

A primeira tarefa do módulo é **apagar código**. Sai o checkout assíncrono v2 que nasceu no módulo de resiliência ([Outbox e checkout assíncrono](./outbox-e-checkout-assincrono.md)): o endpoint que respondia `202 Accepted`, publicava um comando para o próprio ordering e só criava o pedido depois, num consumidor (Listen to Yourself). Depois desta etapa, os endpoints **v1** são o único caminho para criar um pedido.

### Por que remover

A saga orquestrada vai **começar quando o pedido é realizado, dentro da mesma transação que grava o pedido**. Iniciar a saga é gravar algo (o estado da saga, o primeiro comando na outbox) e isso tem de commitar **junto** com o `Order`: ou existem os dois, ou nenhum. Esse momento só existe quando o pedido nasce na própria requisição.

```mermaid
flowchart TD
    subgraph V2["v2 assíncrono (removido)"]
        A1["POST /api/v2/.../orders"] --> A2["tx 1: grava COMANDO na outbox<br/>(o Order ainda não existe)"]
        A2 --> A3["202 Accepted"]
        A2 -.-> A4[("ordering.order.commands")]
        A4 --> A5["consumidor (Listen to Yourself)<br/>tx 2: Order.existing() + orders.add()"]
        A5 --> A6["OrderPlaced na outbox"]
    end
    subgraph V1["v1 síncrono (fica)"]
        B1["POST /api/v1/.../orders"] --> B2["tx única: orders.add(order)<br/>+ OrderPlaced na outbox"]
        B2 --> B3["201 Created"]
        B2 -.->|"aqui a saga orquestrada vai começar,<br/>na MESMA transação"| B4["início da saga"]
    end
```

No v2 haveria dois problemas para a orquestração:

1. **Não existe pedido na transação da requisição.** A primeira transação só grava um comando; o `Order` nasce na segunda, do outro lado de um tópico. Começar a saga na primeira seria começar uma saga para um pedido que ainda pode não existir; começar na segunda exigiria que o consumidor do comando também fosse o ponto de partida da saga.
2. **Dois caminhos de criação = dois pontos de partida.** Com v1 e v2 convivendo, toda regra de início da saga teria de existir duas vezes, com transações diferentes. Um ponto só é mais fácil de garantir.

E a remoção sai barata porque **a corrente de eventos não muda**: o checkout síncrono já publica o `OrderPlaced` pelo `OrderEventListener`, pela outbox, na mesma transação do `orders.add()` (`CheckoutApplicationService.checkout` e `BuyNowApplicationService.buyNow`, ambos `@Transactional`). Billing e catálogo não percebem a diferença.

> **Para fixar:** o 202 do v2 resolvia um problema real (resposta rápida com processamento pesado do outro lado de um tópico). Ele sai porque o desenho seguinte precisa de outra coisa: um ponto de início atômico. Padrão bom no lugar errado vira obstáculo. Foi o que aconteceu aqui.

### O que sai, camada por camada

| Camada | Removido |
|---|---|
| Web | `MyOrderV2Controller`, `OrderAcceptedOutput`; o `POST /api/v2/customers/me/orders` com `application/vnd.order-with-product.v2+json` e `application/vnd.order-with-shopping-cart.v2+json` |
| Gateway | `/api/v2/customers/**` do `Path` da rota do ordering no `gateway-ecommerce` (os caminhos v1 da mesma rota ficam) |
| Aplicação | `CheckoutAsyncApplicationService`, `BuyNowAsyncApplicationService`, `ProcessAcceptedCheckoutApplicationService` |
| Portas de entrada | `ForBuyingWithShoppingCartAsync`, `ForBuyingProductAsync`, `ForProcessingCheckoutAccepted` |
| Comando | `ProcessAcceptedCheckoutIntegrationCommand` |
| Mensageria | `KafkaOrderIntegrationCommandListener`, `KafkaOrderIntegrationCommandPublisher`, `OutboxOrderIntegrationCommandPublisher`, porta `ForPublishingOrderIntegrationCommands` |
| Tópicos | Beans `NewTopic` de `ordering.order.commands` e `ordering.dlt.ordering.order.commands` |
| Configuração | `orderCommandTopicName` e `orderCommandsConsumerGroup` no `AlgaShopMessagingKafkaProperties` e nos YAMLs de ambiente; alias `Ordering.ProcessAcceptedCheckoutIntegrationCommand` no `spring.json.type.mapping` |
| Snapshot | `OrderSnapshot.shoppingCartId`; `OrderSnapshotAssembler.toSnapshot(Order, UUID)`, `toDomain` e os conversores privados que só ele usava |

### O `OrderSnapshot` encolhe

O snapshot tinha duas coisas que só faziam sentido para o consumidor do comando:

- **`shoppingCartId`**: o pedido não é dono do carrinho, mas o consumidor precisava saber qual carrinho esvaziar depois de criar o pedido. Sem consumidor, não há quem esvazie nada depois; o v1 esvazia o carrinho na própria transação.
- **`toDomain`** e os conversores `Billing`/`Shipping`/`Address`/`OrderItem`: reconstruíam o `Order` a partir do JSON, via `Order.existing()`. Só o consumidor recriava pedidos.

O `toSnapshot(Order)` **fica**, agora sem o segundo parâmetro e sem chamador. Ele volta quando a saga montar o conteúdo do comando de emissão de fatura: o snapshot congelado do pedido vai viajar de novo, só que dentro de um comando da saga.

### O que fica, e por quê

Remover demais quebra as aulas seguintes. Cada item abaixo fica por um motivo concreto:

| Fica | Por quê |
|---|---|
| `IntegrationCommand`, `CommandPublishingException` | Ninguém os usa agora, mas os **comandos da saga** vão usá-los. A orquestração fala por comandos |
| `OrderSnapshotAssembler.toSnapshot(Order)` | Volta no comando de emissão de fatura |
| `ForGuardingIdempotency` | O consumidor de eventos de produto depende dela (`runOnce`) |
| `OrderPlacedIntegrationEventAssembler` | O `OrderEventListener` o usa para publicar `OrderPlaced` no checkout síncrono |
| `BillingInputDisassembler`, `ShippingInputDisassembler` | Usados pelos serviços síncronos de checkout e compra direta |
| `CheckoutInput`, `BuyNowInput` | São os corpos que os endpoints v1 recebem |
| A outbox inteira | Continua publicando todos os eventos do ordering; só deixou de transportar o comando de checkout |

### Lições da remoção

- **O compilador só reclama de quem usa uma classe apagada.** Uma classe que ninguém mais usa compila em silêncio. Por isso a remoção segue uma lista, de fora para dentro (controller → aplicação → portas → mensageria → config), e não "apagar até compilar".
- **O compilador não lê YAML.** Um alias de `spring.json.type.mapping` apontando para uma classe que não existe faz o (de)serializer falhar com `IllegalArgumentException` na configuração, e o serviço nem sobe. Depois de apagar classes, procure o nome delas nos arquivos de ambiente. E o último alias da lista não leva vírgula.
- **Remover o bean `NewTopic` não apaga o tópico.** O bean só cria o tópico se ele não existir. `ordering.order.commands` e o DLT dele continuam no broker até alguém excluir pela GUI.
- **O contrato também é código a remover.** Os canais, operações e mensagens do comando saíram do AsyncAPI ([`ordering.yml`](../asyncapi/ordering.yml), [`messages.yml`](../asyncapi/messages.yml)) na mesma etapa, para o contrato não descrever um tópico que não existe mais ([a lição da Fase 40](./asyncapi-contratos.md)). O schema `OrderSnapshot` ficou no `messages.yml`, sem `shoppingCartId` e por enquanto sem referência, pelo mesmo motivo do `toSnapshot(Order)`: ele volta no comando da saga.

### Conferência do que foi feito

Comparando o working tree com a lista do desafio:

| Item | Situação |
|---|---|
| Controller, output, serviços, portas, comando, listener, publishers, porta de saída | ✅ removidos |
| Beans de tópico e DLT, propriedades, alias nos dois YAMLs | ✅ removidos |
| `OrderSnapshot` sem `shoppingCartId`; assembler sem `toSnapshot(Order, UUID)` e `toDomain` | ✅ removidos |
| Itens preservados (tabela acima) | ✅ todos existem |
| Testes Java | ✅ nenhum referencia as classes removidas |
| `OrderSnapshotAssembler` compila | ✅ corrigido: tinha recebido acessores `.value()` que não existem nos records do projeto (`Money(money)`, `Quantity(quantity)`, `Document(document)`…), `order.totalItems()` no lugar de `totalQuantity()` e um argumento a menos no `OrderItemSnapshot`. Voltaram os acessores originais; só a remoção pedida ficou |
| `KafkaStockIntegrationEventListener` compila | ✅ corrigido: tirado o import de `ProcessAcceptedCheckoutIntegrationCommand` (e os outros imports sem uso) |
| Gateway sem `/api/v2/customers/**` | ✅ removido do `Path` da rota do ordering; os caminhos v1 e o `CircuitBreaker` ficaram |
| Comentários | ✅ `OrderSnapshot.java`, `OrderSnapshotAssembler.java` e `KafkaConfig.java` não falam mais do comando |
| Testes do ordering | ✅ `./gradlew test`: 304 testes, 0 falhas, nenhum teste Java alterado |
| Ambiente | ⚠️ tópicos antigos continuam no broker; conferir se a tabela `outbox` não tem linhas pendentes para `ordering.order.commands` antes de subir |

### Roteiro de verificação

1. Testes do ordering passando, sem alterar nenhum teste Java.
2. Checkout v1 com carrinho e compra direta v1, autenticado como o cliente `6e148bd5-47f6-4022-b9da-07cfaa294f7a`, com o Visa final 1022 (`creditCardId` `0199deda-9486-7d27-b40c-761f561a24d8`, pago na captura). Na GUI do Kafka, cada pedido percorre os mesmos seis eventos do módulo 8 e termina em `Ordering.OrderReadyIntegrationEvent`, com `ready_at` preenchido.
3. `POST /api/v2/customers/me/orders` autenticado, pelo gateway e direto no ordering, com `Content-Type: application/vnd.order-with-shopping-cart.v2+json`: as duas respostas são `404 Not Found`. O media type certo garante que o 404 prova que o caminho saiu, e não que o `Content-Type` estava errado.
4. Nenhuma mensagem nova em `ordering.order.commands`; o consumer group `ordering.order-commands` sem membros ativos.

---

## Removendo a saga coreografada

O segundo desafio apaga a [saga coreografada](./saga-coreografada.md) inteira: a corrente de seis eventos que levava o pedido de `PLACED` a `READY`, em que cada serviço reagia ao evento do anterior e decidia sozinho o próximo passo.

### Por que remover antes de construir

Na orquestração, **quem decide o próximo passo é o ordering**: ele manda um comando a um participante, espera a resposta e decide o que vem depois. Se os consumidores da coreografia continuassem de pé, o mesmo pedido seria conduzido por **dois caminhos ao mesmo tempo**: o billing geraria a fatura ao ver o `OrderPlaced` **e** quando recebesse o comando do orquestrador; o catálogo reservaria estoque ao ver o `OrderPaid` **e** pelo comando. Fatura dupla, reserva dupla, e dois "donos" do fluxo brigando pelo estado do pedido.

Por isso a coreografia sai antes de a saga entrar, e o sistema passa por um momento proposital em que **nada conduz o pedido**:

```mermaid
flowchart LR
    subgraph ANTES["Antes: coreografia (removida)"]
        direction LR
        P1["OrderPlaced"] --> B1["billing gera fatura"] --> I1["InvoiceIssued"] --> B2["billing cobra"] --> I2["InvoicePaid"] --> O1["ordering: PAID"] --> P2["OrderPaid"] --> C1["catálogo reserva"] --> S1["StockReservationConfirmed"] --> O2["ordering: READY"]
    end
    subgraph AGORA["Agora: ninguém conduz"]
        direction LR
        Q1["checkout v1"] --> Q2["Order PLACED<br/>+ OrderPlaced na outbox"] --> Q3["ordering.order.events"]
        Q3 -.->|"nenhum consumidor"| Q4["o pedido fica em PLACED"]
    end
```

> **Para fixar:** o pedido parado em `PLACED` não é bug, é o **estado esperado** entre a demolição e a construção. Ele prova que nenhum consumidor esquecido continua empurrando o fluxo, o que é exatamente o risco de deixar os dois mecanismos convivendo.

### O que sai, serviço por serviço

| Serviço | Consumidores | Eventos e apoio | Configuração |
|---|---|---|---|
| **ordering** | `KafkaInvoiceIntegrationEventListener`, `KafkaStockIntegrationEventListener` | `InvoicePaid`, `InvoiceCanceled`, `StockReservationConfirmedIntegrationEvent`; `OrderManagementApplicationService` e a porta `ForManagingOrders` (só os consumidores usavam); `OrderManagementApplicationServiceIT` | Tópicos e grupos de fatura e estoque, os dois DLTs, três aliases |
| **billing** | `KafkaOrderIntegrationEventListener` (gerava a fatura), `KafkaStockIntegrationEventListener` (estornava) | `OrderPlacedIntegrationEvent`, os seis snapshots de `application.order.snapshot`, `GenerateInvoiceInputAssembler`, `StockReservationRejectedIntegrationEvent` | Tópicos e grupos de pedido e estoque, os dois DLTs, dois aliases |
| **catálogo** | `KafkaOrderIntegrationEventListener` (reservava o estoque) | `OrderPaidIntegrationEvent`, `OrderItemSnapshot`, o `OrderPaidIntegrationEventAssembler` **do catálogo**; `StockReservationEventListener`, `StockIntegrationEventPublisher` (interface e bean), os eventos de integração **e de domínio** da reserva | Bean do tópico `product-catalog.stock.events`, DLT do tópico de pedidos, propriedades, três aliases |

Depois disso, o ordering só consome eventos de **produto**, o billing só conhece o tópico de **fatura**, e o catálogo não consome nada.

### O que fica, e por quê

| Fica | Onde | Por quê |
|---|---|---|
| `OrderEventListener` e os eventos `OrderPlaced/Paid/Canceled/Ready` | ordering | O pedido continua anunciando o que acontece com ele. Hoje só o `OrderPlaced` sai, porque nada mais muda o estado; quando a saga voltar a chamar `markAsPaid`, `markAsReady` e `markAsCanceled`, os outros voltam a sair sozinhos |
| `OrderPaidIntegrationEventAssembler` **do ordering** | ordering | O `OrderEventListener` o usa para publicar o pedido pago (o homônimo do catálogo saiu) |
| `InvoiceManagementApplicationService.generate` e `refundByOrderId`, com `GenerateInvoiceInput` | billing | Viram os alvos dos **comandos** "emita a fatura" e "estorne" |
| `StockManagementApplicationService.reserveForOrder`, inputs, `DuplicateStockReservationException`, `existsByOrderId` | catálogo | A reserva em si não muda: só sai quem a chamava. O comando "reserve o estoque" vai chamá-la |
| `createDeadLetterTopic` no `KafkaConfig` do catálogo | catálogo | Fica sem uso, de propósito: os tópicos de comando das próximas aulas voltam a usá-lo |

Note a regra que separa os dois lados: **sai quem reage e quem anuncia para outro serviço reagir; fica quem faz o trabalho**. Gerar fatura, estornar e reservar estoque continuam existindo. Só muda quem os aciona: um evento interpretado por cada serviço → um comando enviado pelo orquestrador.

### A `StockReservation` deixa de anunciar

```java
// antes
public class StockReservation extends AbstractAggregateRoot<StockReservation> {
    public static StockReservation confirmed(...) {
        StockReservation r = new StockReservation(orderId, Status.CONFIRMED, items);
        r.registerEvent(StockReservationConfirmedEvent.builder()...build());
        return r;
    }
// agora
public class StockReservation {
    public static StockReservation confirmed(...) {
        return new StockReservation(orderId, Status.CONFIRMED, items);
    }
```

Na coreografia, a reserva **anunciava** o resultado para quem quisesse ouvir (o ordering ouvia a confirmação, o billing ouvia a rejeição). Na orquestração, o resultado vira a **resposta ao comando**, endereçada ao orquestrador. O evento de domínio que existia só para virar evento de integração perde a razão de existir, e a reserva volta a ser um documento simples.

### Um detalhe que ficou: o billing ainda cobra a própria fatura

O `KafkaInvoiceIntegrationEventListener` do billing continua consumindo o próprio `InvoiceIssued` para chamar `processPayment` (o Listen to Yourself da [aula 7.8](./workflow-coreografado.md#78--passo-4-o-billing-cobra-a-própria-fatura-e-o-webhook-fecha-o-ciclo)). Ele não está na lista de remoção. Hoje fica **ocioso**, porque nada mais gera fatura; volta a trabalhar quando o comando de emissão da saga chamar o `generate`. Vale observar nas próximas aulas se ele continua ou se a cobrança também vira um comando do orquestrador.

### Contratos

Os contratos AsyncAPI seguiram o código na mesma etapa:

- [`ordering.yml`](../asyncapi/ordering.yml): só consome `product-catalog.product.events` e publica `ordering.order.events`.
- [`billing.yml`](../asyncapi/billing.yml): só `billing.invoice.events` (publica, consome o próprio `InvoiceIssued`, e o DLT).
- [`product-catalog.yml`](../asyncapi/product-catalog.yml): só publica `product-catalog.product.events`.
- [`messages.yml`](../asyncapi/messages.yml): saíram as mensagens, headers e payloads de `StockReservationConfirmed/Rejected`, o trait `KafkaStockReservationKey` e o schema `StockReservationId`. A divergência de key que a [saga coreografada](./saga-coreografada.md#o-contrato-diz-uma-key-o-código-usa-outra) apontava desaparece junto.

### Conferência

| Item | Situação |
|---|---|
| Ordering: consumidores, três eventos, serviço, porta, IT, propriedades, DLTs, aliases | ✅ removidos; último alias sem vírgula nos dois YAMLs |
| Billing: consumidores, evento, seis snapshots, assembler, evento de estoque, propriedades, DLTs, aliases | ✅ removidos; imports órfãos no listener de fatura limpos |
| Catálogo: consumidor, evento, snapshot, assembler, publicador, eventos de integração e de domínio, tópico, DLT, propriedades, aliases | ✅ removidos; `StockReservation` sem `AbstractAggregateRoot` (o import que sobrou foi tirado) |
| Itens preservados | ✅ todos existem |
| Sobras em `src/` dos três serviços | ✅ nenhuma referência às classes e propriedades removidas |
| Testes unitários (`./gradlew test`) | ✅ ordering 304, billing 48, catálogo 117 — 0 falhas |
| Testes de contrato | ✅ ordering 12, catálogo 17 — 0 falhas |
| Testes de integração do ordering | ✅ 185, 0 falhas (já sem o `OrderManagementApplicationServiceIT`) |
| Testes de integração do billing | ✅ 20, 0 falhas — depois de corrigir três problemas anteriores a este desafio (abaixo) |
| Testes de integração do catálogo | ✅ 12, 0 falhas — idem |

### Os testes que já estavam quebrados

O primeiro `./gradlew check` depois da remoção falhou em 20 ITs do billing e 11 do catálogo. Nenhum por causa da remoção: os ITs desses dois serviços não rodavam com sucesso desde o módulo 7. Foram quatro causas:

| Serviço | Sintoma | Causa | Correção |
|---|---|---|---|
| billing | Contexto não sobe (20 ITs) | O `application-test-env.yaml` nunca ganhou Kafka: faltavam `algashop.messaging.kafka.*` (`@NotBlank`) e o `spring.json.type.mapping` que o `KafkaConsumerTypeIdIdentifier` lê por `@Value` | Bloco `spring.kafka` **no padrão do ordering** (mapping, producer, consumer) + tópico e grupo de fatura. Diferença do ordering: `spring.kafka.listener.auto-startup: false`, para o listener do `InvoiceIssued` não entrar no grupo `billing.invoice-events` do cluster de desenvolvimento durante os testes |
| billing | `shouldProcessInvoicePaymentAndCancelInvoice` | O teste verificava a chamada ao `InvoiceEventListener`, que virou `@TransactionalEventListener` (AFTER_COMMIT) na aula 8.3. O teste é `@Transactional` com rollback: o commit nunca acontece e o listener nunca roda | O teste passou a verificar o que de fato é responsabilidade do agregado: o `InvoiceCanceledEvent` publicado no save (`ApplicationEvents`, que a classe já registrava) |
| billing | `PaymentGatewayServiceFastpayImplIT` passa sozinho, falha na suíte | Cada classe parava e subia o WireMock na porta 8788; o contexto Spring em cache guardava no pool uma conexão com o servidor anterior → `NoHttpResponseException` | Um WireMock só para a JVM: `startMock` não sobe de novo se já estiver rodando, `stopMock` não para. O reset de cenários do `@BeforeEach` continua isolando os testes |
| catálogo | `StockTransactionIT` não sobe | O `KafkaConsumerTypeIdIdentifier` (aula 8.7) exige o mapping, ausente na configuração de teste | Bloco `spring.kafka` no `application-test-env.yml`, no padrão do ordering |
| catálogo | 7 ITs de `QuantityInStockAdjustment` | `@Transactional(propagation = MANDATORY)` (aula 8.5) recusa a chamada direta dos testes de fatia e de concorrência | `MANDATORY` removido (ver abaixo) |

**Por que tirar o `MANDATORY` em vez de abrir transação nos testes.** O ajuste é um `findAndModify` condicional: já é atômico por documento, sem transação. Quando o chamador abre uma (`restock`, `withdraw`, `reserveForOrder`, todos `@Transactional`), o `MongoOperations` participa dela e o ajuste commita junto com o movimento, com ou sem a anotação. O `MANDATORY` era só uma trava contra chamadas sem transação. Envolver cada chamada do teste de concorrência numa transação Mongo, ao contrário, mudaria o que ele testa: transações concorrentes no mesmo documento disputam com `WriteConflict` em vez de exercitar a atomicidade do `findAndModify`. Perdeu-se a trava; ficou a atomicidade, e os testes Java do catálogo não mudaram.

### Roteiro de verificação

1. Subir os três serviços e ver que iniciam sem erro (um alias apontando para classe apagada derruba a subida com `IllegalArgumentException`).
2. Checkout v1 com carrinho, autenticado como o cliente `6e148bd5-47f6-4022-b9da-07cfaa294f7a`, Visa final 1022 (`creditCardId` `0199deda-9486-7d27-b40c-761f561a24d8`). O pedido deve **parar em `PLACED`**:
   - em `ordering.order.events` chega só `Ordering.OrderPlacedIntegrationEvent`; nada novo em `billing.invoice.events` nem em `product-catalog.stock.events`;
   - `select id, status, placed_at, paid_at, ready_at from public."order" where id = '<orderId>';` (banco `ordering`): `PLACED`, `paid_at` e `ready_at` vazios;
   - `select id, status from public.invoice where order_id = '<orderId>';` (banco `billing`): nenhuma linha;
   - `db.stock_reservations.find({ orderId: "<orderId>" })` (mongosh, catálogo): nenhum documento.
3. Consumer groups na GUI do Kafka:

| Consumer group | Esperado |
|---|---|
| `ordering.invoice-events` | sem membros ativos |
| `ordering.stock-events` | sem membros ativos |
| `billing.order-events` | sem membros ativos |
| `billing.stock-events` | sem membros ativos |
| `product-catalog.order-events` | sem membros ativos |
| `ordering.product-events` | continua ativo |

4. Limpeza opcional do broker (remover o bean não apaga o tópico): `product-catalog.stock.events`, `ordering.dlt.billing.invoice.events`, `ordering.dlt.product-catalog.stock.events`, `billing.dlt.ordering.order.events`, `billing.dlt.product-catalog.stock.events`, `product-catalog.dlt.ordering.order.events`.

---

## Modelando a saga: `PlaceOrderSaga`

Com o terreno limpo, a primeira peça da orquestração é **o estado da saga**: o registro que responde "em que passo está o pedido 42?", a pergunta que a coreografia não sabia responder. Por enquanto é só o modelo, sem persistência, sem comandos e sem métodos de transição.

| Arquivo | Pacote | Papel |
|---|---|---|
| `SagaStatus` | `core/application/saga` | Ciclo de vida **genérico** de qualquer saga |
| `SagaTransitionException` | `core/application/saga` | Erro de transição inválida, com fábricas por tipo de violação |
| `PlaceOrderSagaStep` | `core/application/order/saga` | Os passos **desta** saga, a de realizar pedido |
| `PlaceOrderSaga` | `core/application/order/saga` | O estado de uma execução: `sagaId`, `orderId`, `status`, `step`, `version` |

### Por que em `application`, e não no domínio

O `Order` é domínio: tem regras de negócio (o que pode ser pago, cancelado, aprovado). A saga **coordena** serviços: lembra qual comando foi enviado e qual resposta se espera. Isso é orquestração de caso de uso, o mesmo nível dos application services. Por isso `PlaceOrderSaga` mora em `core/application/order/saga`, ao lado do que vai usá-la, e o que é reaproveitável por qualquer saga futura (`SagaStatus`, `SagaTransitionException`) fica um nível acima, em `core/application/saga`.

### Duas máquinas de estado: status × passo

A saga guarda duas coisas diferentes, e cada uma é uma máquina de estados própria:

- **Status**: *em que fase da vida* a saga está. Genérico, igual para qualquer saga.
- **Passo**: *onde* no fluxo desta saga ela está. Específico do pedido.

```mermaid
stateDiagram-v2
    direction LR
    [*] --> RUNNING
    RUNNING --> SUCCEEDED: último passo deu certo
    RUNNING --> COMPENSATING: um passo falhou
    COMPENSATING --> COMPENSATED: compensações concluídas
    SUCCEEDED --> [*]
    COMPENSATED --> [*]
```

```mermaid
stateDiagram-v2
    [*] --> PLACING_ORDER
    PLACING_ORDER --> INVOICING: Issue Invoice
    INVOICING --> RESERVING_STOCK: Invoice Paid
    RESERVING_STOCK --> APPROVING_ORDER: Reservation Confirmed
    INVOICING --> CANCELLING_ORDER: Invoice Canceled
    RESERVING_STOCK --> REFUNDING_INVOICE: Reservation Rejected
    REFUNDING_INVOICE --> CANCELLING_ORDER: estorno feito
```

Cada passo corresponde a uma linha do [mapa detalhado](#o-mapa-da-orquestração):

| Passo | Linha do mapa | Status em que aparece |
|---|---|---|
| `PLACING_ORDER` | Placed Order (início, na transação do checkout) | `RUNNING` |
| `INVOICING` | Invoicing (comando `Issue Invoice`) | `RUNNING` |
| `RESERVING_STOCK` | Reserving Stock, o **pivô** (comando `Reserve Stock`) | `RUNNING` |
| `APPROVING_ORDER` | Approving order (comando `Approve order`) | `RUNNING` → `SUCCEEDED` |
| `REFUNDING_INVOICE` | Compensação `Cancel invoice` (a fatura já está paga, então cancelar é estornar) | `COMPENSATING` |
| `CANCELLING_ORDER` | Compensação `Cancel order` | `COMPENSATING` → `COMPENSATED` |

Dois caminhos levam ao cancelamento do pedido, e o enum declara os dois: `CANCELLING_ORDER(INVOICING, REFUNDING_INVOICE)`. Se a fatura é recusada, não há o que estornar e a saga cancela direto. Se o estoque é rejeitado, ela estorna primeiro. É a ordem inversa das compensações, escrita em um lugar só.

### O padrão `previousStatus`, de novo

Os dois enums usam o mesmo truque do `OrderStatus` do domínio: **cada estado declara de onde pode vir**.

```java
public enum PlaceOrderSagaStep {
    PLACING_ORDER,
    INVOICING(PLACING_ORDER),
    RESERVING_STOCK(INVOICING),
    APPROVING_ORDER(RESERVING_STOCK),
    REFUNDING_INVOICE(RESERVING_STOCK),
    CANCELLING_ORDER(INVOICING, REFUNDING_INVOICE);
    ...
    public boolean canChangeTo(PlaceOrderSagaStep newStatus) {
        return newStatus.previousStatus.contains(this);
    }
}
```

A regra de transição fica junto da definição do estado, e não espalhada em `if`s. Para saber de onde se chega em `CANCELLING_ORDER`, basta ler a linha dele.

### `PlaceOrderSaga`: o estado de uma execução

```java
public static PlaceOrderSaga start(OrderId orderId)      // RUNNING em PLACING_ORDER, version 0
public static PlaceOrderSaga existing(UUID sagaId, OrderId orderId,
                                      SagaStatus status, PlaceOrderSagaStep step, long version)
```

- **`start` × `existing`**: a mesma separação do `Order` (`draft` para nascer, `existing` para reidratar do banco). Só `start` decide o estado inicial; `existing` aceita o que vier persistido.
- **`version`**: preparada para lock otimista. Quando duas respostas do mesmo pedido chegarem ao mesmo tempo, a segunda gravação vai falhar em vez de sobrescrever a primeira.
- **Consultas `isRunningAt(step)`, `isCompensatingAt(step)`, `isActive()`**: é com elas que o orquestrador vai decidir se uma resposta ainda é esperada. Um `Invoice Paid` só faz sentido se a saga está `RUNNING` em `INVOICING`; se chegar de novo (reentrega), a saga já estará em `RESERVING_STOCK` e a resposta pode ser ignorada com segurança. É a base da idempotência das replies, aquilo que a coreografia resolvia mandando duplicatas para o DLT.
- **Guardas privadas `require`, `changeStatusTo`, `changeStepTo`**: validam pré-condição, status e passo, lançando `SagaTransitionException`. Ainda não têm chamador; os métodos de transição das próximas aulas vão usá-las.

### Revisão desta etapa

- **Corrigido**: `APPORVING_ORDER` → `APPROVING_ORDER`. O enum vai ser gravado como texto na tabela da saga, e renomear um valor depois de persistido exige migração de dados.
- **Corrigido**: import sem uso de `OrderStatus` em `SagaStatus`.
- **Para ficar de olho**: status e passo são validados **separadamente**. Os enums sozinhos não impedem combinações sem sentido, como `COMPENSATING` em `APPROVING_ORDER` ou `RUNNING` em `REFUNDING_INVOICE`. Quem vai garantir a coerência entre os dois são os métodos de transição (cada um mudando passo e status juntos, com `require` antes). Um teste unitário por transição é o lugar natural para travar isso.

---

## Persistindo a saga

O modelo da seção anterior só existia em memória. Esta etapa dá a ele uma tabela, sem mudar uma linha do `PlaceOrderSaga`: tudo o que é banco fica na infraestrutura.

```mermaid
flowchart LR
    A["PlaceOrderSaga<br/>(core/application)"] -->|"porta de saída"| B["ForStoringPlaceOrderSagas<br/>ofId · add"]
    B -.->|"implementada por"| C["PlaceOrderSagasPersistenceProvider<br/>toEntity · toSaga"]
    C --> D["PlaceOrderSagaPersistenceEntityRepository<br/>(Spring Data JPA)"]
    D --> E[("saga_instance")]
```

É o mesmo desenho de [ports & adapters](../01-arquitetura-design/ports-hexagonal.md) do `Orders`: a aplicação conhece a porta `ForStoringPlaceOrderSagas` (`core/ports/out/order/saga`), e o adaptador traduz entre o objeto da aplicação e a entidade JPA.

### A classe abstrata `SagaInstancePersistenceEntity`

O centro da etapa é uma entidade **abstrata** que serve de base para todas as sagas do serviço, presentes e futuras:

```mermaid
classDiagram
    class SagaInstancePersistenceEntity {
        <<abstract>>
        UUID sagaId
        String aggregateId
        SagaStatus status
        OffsetDateTime createdAt
        OffsetDateTime updatedAt
        long version
    }
    class PlaceOrderSagaPersistenceEntity {
        PlaceOrderSagaStep step
    }
    SagaInstancePersistenceEntity <|-- PlaceOrderSagaPersistenceEntity : saga_type = PLACE_ORDER
```

Como ela foi construída, peça por peça:

| Peça | O que faz | Por quê |
|---|---|---|
| `abstract` + `@Entity` | É uma entidade JPA, mas não pode ser instanciada | Não existe "saga genérica" no banco: toda linha é de um tipo concreto |
| `@Inheritance(SINGLE_TABLE)` | Base e subclasses gravam na **mesma** tabela | Uma consulta por status atende qualquer tipo de saga, sem join |
| `@DiscriminatorColumn(name = "saga_type")` + `@DiscriminatorValue("PLACE_ORDER")` na subclasse | Cada linha diz de que tipo é | Ao ler a linha, o Hibernate sabe qual subclasse instanciar |
| Campos na base: `sagaId`, `aggregateId`, `status` | O que **toda** saga tem | Identidade, o agregado que ela conduz e o ciclo de vida genérico (`SagaStatus`) |
| Campo na subclasse: `step` | O que é **desta** saga | Cada saga tem seu próprio enum de passos |
| `@CreatedDate`, `@LastModifiedDate` + `AuditingEntityListener` | Datas preenchidas pelo Spring Data | A mesma auditoria das outras entidades, com o `DateTimeProvider` de `OffsetDateTime` do projeto |
| `@Version` | Lock otimista | Duas respostas do mesmo pedido gravando ao mesmo tempo: a segunda falha em vez de sobrescrever |
| `aggregateId` como `String` | Serve a qualquer agregado | Hoje guarda o TSID do pedido; amanhã pode ser outro id |

**Por que `SINGLE_TABLE` e não as outras estratégias.** `JOINED` separaria os campos comuns e os específicos em tabelas diferentes, com join a cada leitura. `TABLE_PER_CLASS` daria uma tabela por saga, e "quais sagas estão ativas?" viraria um `UNION`. Com single table, o preço é que colunas específicas de uma saga ficam vazias nas linhas das outras. Com poucos campos por saga, é um preço baixo.

### A tabela `saga_instance` (V010)

| Coluna | Origem | Observação |
|---|---|---|
| `saga_id` | base | PK |
| `saga_type` | discriminador | `PLACE_ORDER` |
| `aggregate_id` | base | id do pedido |
| `status` | base | `RUNNING`, `COMPENSATING`, `SUCCEEDED`, `COMPENSATED` |
| `step` | subclasse | `not null`: toda saga do serviço precisa ter passo |
| `failure` | — | reservada, ainda sem campo na entidade |
| `created_at`, `updated_at` | auditoria | |
| `version` | `@Version` | |

Duas decisões no SQL valem a leitura:

- **`unique (saga_type, aggregate_id)`**: uma saga de realizar pedido por pedido. Se o início da saga for executado duas vezes para o mesmo pedido, o banco recusa a segunda.
- **Índice parcial `idx_saga_instance_active`** em `(saga_type, status)` `where status in ('RUNNING', 'COMPENSATING')`: só as sagas ativas entram no índice. As terminadas (a grande maioria, com o tempo) não pesam. É exatamente o conjunto que um job de recuperação ou de timeout vai varrer.

### O adaptador

`PlaceOrderSagasPersistenceProvider` converte nos dois sentidos:

- `toEntity(saga)`: cria a `PlaceOrderSagaPersistenceEntity` e copia `sagaId`, `orderId` (como texto), `status`, `step` e `version`.
- `toSaga(entity)`: reidrata com `PlaceOrderSaga.existing(...)`, a fábrica que a seção anterior preparou para isso.

A classe é `@Transactional(readOnly = true)` por padrão, e só o `add` abre escrita.

### Revisão desta etapa

- **Corrigido**: o campo se chamava `updateAt`, mas a coluna é `updated_at`. A estratégia de nomes do Spring Boot transformaria o campo em `update_at`, e como o projeto usa `ddl-auto: none`, nada acusaria na subida: o primeiro `insert` de uma saga falharia com "column update_at does not exist". Renomeado para `updatedAt`. Nenhum teste grava saga ainda, por isso a suíte não pegou.
- **Corrigido**: import sem uso (`LastModifiedBy`).
- **Para quando as transições chegarem**: o `add` monta uma entidade nova a cada chamada. Para inserir, funciona. Para **atualizar** uma saga já gravada, dois problemas vão aparecer:
  1. a entidade nova chega com `createdAt` nulo, o `merge` copia o nulo por cima e viola o `not null`;
  2. a `version` incrementada pelo banco não volta para o objeto `PlaceOrderSaga`, e a gravação seguinte, com a versão velha, daria `OptimisticLockException`.

  O `OrderPersistenceProvider` já resolve os dois: `findById` decide entre `update` e `insert`, a atualização copia os campos para a entidade carregada, e o `updateVersion` devolve a versão ao agregado por reflexão. É o padrão a reaproveitar.
- **Testes**: `./gradlew check` do ordering verde (304 unitários, 12 de contrato, 185 de integração, com a V010 aplicada no Postgres de teste). Um IT do `PlaceOrderSagasPersistenceProvider` (gravar, ler, gravar de novo) é o que pegaria os itens acima.

---

## Onde a saga começa

A primeira seção deste doc explicou **por que** o checkout assíncrono saiu: a saga precisa nascer na mesma transação que grava o pedido. Esta etapa é o **como**.

```mermaid
sequenceDiagram
    participant API as POST /api/v1/.../orders
    participant CK as CheckoutApplicationService<br/>(@Transactional)
    participant L as OrderEventListener
    participant SC as PlaceOrderSagaCoordinator
    participant DB as Postgres (uma transação)
    API->>CK: checkout(input)
    CK->>DB: orders.add(order)
    Note over CK,L: o save publica OrderPlacedEvent
    L->>DB: onOrderPlacedPublishIntegrationEvent<br/>→ linha na outbox (OrderPlaced)
    L->>SC: onOrderPlacedStartSaga → start(orderId)
    SC->>DB: PlaceOrderSaga.start → saga_instance (RUNNING, PLACING_ORDER)
    CK-->>API: 201 Created
    Note over DB: commit único: pedido + outbox + saga,<br/>ou nenhum dos três
```

As peças:

| Peça | Onde | Papel |
|---|---|---|
| `ForCoordinatingPlaceOrderSaga.start(rawOrderId)` | `core/ports/in/order/saga` | Porta de **entrada** do orquestrador |
| `PlaceOrderSagaCoordinator` | `core/application/order/saga` | Implementação: cria a saga com `PlaceOrderSaga.start` e grava pela porta de saída `ForStoringPlaceOrderSagas` |
| `OrderEventListener.onOrderPlacedStartSaga` | `infrastructure/adapters/in/listerner/order` | Adaptador de entrada: traduz o evento de domínio em chamada à porta |

Três detalhes que fazem a diferença:

- **Dois ouvintes para o mesmo evento.** O antigo `listen(OrderPlacedEvent)` virou `onOrderPlacedPublishIntegrationEvent` e ganhou um irmão, `onOrderPlacedStartSaga`. Nomes que dizem *o que* cada um faz são mais legíveis que dois `listen` sobrecarregados, e cada responsabilidade pode evoluir sozinha.
- **Os dois são `@EventListener` síncronos**, e não `@TransactionalEventListener`. Rodam *dentro* da transação do checkout, e o `@Transactional` do coordenador apenas participa dela. Se gravar a saga falhar, o checkout inteiro volta: não existe pedido sem saga. É o mesmo raciocínio da outbox: com escrita local, publicar dentro da transação é o objetivo.
- **Uma saga por pedido** fica garantida pela constraint `unique (saga_type, aggregate_id)` da V010.

O efeito colateral mais visível: todo checkout v1 agora grava uma linha em `saga_instance`. O `CheckoutApplicationServiceIT` passou a exercitar a persistência da saga, e foi ele que confirmou, no Postgres de teste, a correção do `updatedAt` da seção anterior. O teste também foi ajustado para verificar os dois ouvintes.

> **Para fixar:** a saga começa e... para. O coordenador ainda não envia o comando `Issue Invoice`. Um pedido novo fica `PLACED`, com a saga `RUNNING` em `PLACING_ORDER`. Agora há um registro dizendo onde ele parou, o que a coreografia nunca teve.

## Entrada × saída: as interfaces de integração

O mesmo nome, `IntegrationEvent`, servia para duas coisas opostas: o evento que o serviço **publica** e o que ele **consome**. No consumidor, o `getAggregateId()` existia só por simetria, e nenhum código o chamava. Com comandos e respostas chegando, a ambiguidade ia piorar. A interface foi separada por **direção**:

| Interface | Direção | O que exige | Quem implementa |
|---|---|---|---|
| `OutboundIntegrationEvent` | sai do serviço | `getAggregateId()` (vira a **key**) e `getIdempotencyKey()` (default `null`) | ordering: eventos de pedido; billing: de fatura; catálogo: de produto |
| `InboundIntegrationEvent` | entra no serviço | nada (marcador) | ordering: os eventos de produto que consome |
| `OutboundIntegrationCommand` | sai do serviço | `getAggregateId()` e `getIdempotencyKey()` (agora default `null`) | os comandos da saga, quando existirem |

A regra por trás: **quem publica escolhe a key**. No consumo, a key já chega pronta no header `KafkaHeaders.RECEIVED_KEY`, e o evento de entrada não precisa saber calculá-la. Por isso os eventos de produto do ordering perderam o `getAggregateId()`.

As portas e os adaptadores acompanharam nos três serviços (`ForPublishingOrderIntegrationEvents`, `InvoiceIntegrationEventPublisher`, `ProductIntegrationEventPublisher`, os publishers Kafka e de outbox, as `*PublishingException`). Billing e catálogo fizeram só o rename, porque hoje apenas publicam.

## O contrato da orquestração

Os contratos AsyncAPI foram reescritos **antes** do código dos comandos (contract-first): o desenho está escrito e validado, e as próximas aulas o implementam.

| Serviço | Publica | Consome |
|---|---|---|
| ordering | `ordering.order.events`; comandos em `billing.invoice.commands` (`IssueInvoice`, `CancelInvoice`) e `product-catalog.stock.commands` (`ReserveStock`) | `product-catalog.product.events`; respostas em `ordering.saga.replies` (grupo `ordering.saga-replies`) |
| billing | `billing.invoice.events`; respostas no tópico indicado pelo comando | `billing.invoice.commands` (grupo `billing.invoice-commands`); o próprio `InvoiceIssued` |
| catálogo | `product-catalog.product.events`; respostas no tópico indicado pelo comando | `product-catalog.stock.commands` (grupo `product-catalog.stock-commands`) |

Cada tópico consumido tem seu DLT (`ordering.dlt.ordering.saga.replies`, `billing.dlt.billing.invoice.commands`, `product-catalog.dlt.product-catalog.stock.commands`).

### Request-reply sobre Kafka

```mermaid
sequenceDiagram
    participant O as ordering (orquestrador)
    participant B as billing
    O->>B: IssueInvoice (billing.invoice.commands)<br/>headers: kafka_correlationId, kafka_replyTopic=ordering.saga.replies
    Note over B: emite e cobra a fatura
    B-->>O: InvoicePaidReply ou InvoiceCanceledReply<br/>(no tópico de kafka_replyTopic, mesmo kafka_correlationId)
```

Os headers carregam o mecanismo:

| Header | Em | Para quê |
|---|---|---|
| `kafka_replyTopic` | comando | Diz **para onde** responder. O participante não precisa conhecer o orquestrador: responde onde o comando mandou. No contrato do billing e do catálogo, o `reply.address` aponta para `$message.header#/kafka_replyTopic` |
| `kafka_correlationId` | comando **e** resposta | Liga a resposta ao comando que a originou |
| `idempotency-key` | comando | Presente quando o comando sai pela outbox |

São os nomes que o Spring Kafka usa em `KafkaHeaders.REPLY_TOPIC` e `KafkaHeaders.CORRELATION_ID`. Todas as mensagens novas usam `key = orderId` (trait `KafkaOrderKey`): comandos e respostas do mesmo pedido ficam na mesma partição, em ordem.

### Eventos × respostas

O billing passa a ter **duas** saídas para o mesmo fato:

| | `InvoicePaidEvent` (evento) | `InvoicePaidReply` (resposta) |
|---|---|---|
| Destinatário | ninguém em particular | o orquestrador que mandou o comando |
| Tópico | `billing.invoice.events` | o de `kafka_replyTopic` |
| Faz a saga andar? | não | sim |

É a separação que o [mapa detalhado](#o-mapa-da-orquestração) mostra nas colunas *Reply* e *Success/Failure Event*, agora escrita no contrato.

### Revisão do contrato

- ~~**Corrigido**: `CancelInvoiceCommand` passou a `Ordering.`~~ **Revertido na etapa "O billing executa o comando"**: a convenção do projeto para comandos é prefixo = serviço que **executa** (`Billing.CancelInvoiceIntegrationCommand`). O prefixo do produtor vale para eventos. Ver [a convenção dos `__TypeId__`](#a-convenção-dos-__typeid__).
- **Não há comando para `Approve order`**: aprovar é um passo interno do ordering, que não sai do serviço.
- **O contrato está à frente do código**: nenhum comando nem resposta existe em Java ainda. Quando existirem, os `__TypeId__` do contrato têm de bater com os aliases do `spring.json.type.mapping`, e é aí que um contrato sem validação costuma começar a mentir ([Fase 40](./asyncapi-contratos.md)).
- `$ref` dos quatro arquivos: nenhum quebrado.

### Revisão do código

- Comentários corrigidos: as `OutboundIntegrationEvent` de billing e catálogo diziam "eventos de integração **do order**" (copiado do ordering); a do catálogo tinha perdido a explicação de key → partição → ordem por agregado; a do ordering ainda descrevia o lado consumidor. O `InboundIntegrationEvent` ganhou comentário próprio.
- Testes: ordering `check` verde (304 unitários, 12 de contrato, 185 de integração); billing e catálogo compilam e os unitários passam.

---

## A outbox aprende a transportar comandos

O orquestrador vai mudar o estado da saga **e** enviar um comando a cada passo: "saga agora em `INVOICING`" + "billing, emita a fatura". São duas escritas, uma no Postgres e outra no Kafka: o mesmo **dual-write** que a [outbox da Fase 41](./outbox-e-checkout-assincrono.md) resolveu para eventos. A saída é a mesma: o comando também passa pela outbox, gravado na transação que muda a saga. Se a transição commitar, o comando sai em algum momento; se ela voltar, o comando nunca existiu.

A outbox, porém, foi escrita para eventos. Esta etapa a prepara para levar as duas coisas.

### Evento × comando na mesma tabela

Um comando precisa de duas informações que um evento não tem: **a que conversa a resposta pertence** e **para onde respondê-la**. Viraram duas colunas novas, opcionais:

| Coluna | Evento (`OrderPlaced`) | Comando (`IssueInvoice`, em breve) | Vira o header |
|---|---|---|---|
| `channel_name` | `ordering.order.events` | `billing.invoice.commands` | (tópico do record) |
| `aggregate_id` | id do pedido | id do pedido | (key do record) |
| `message_type` | `Ordering.OrderPlacedIntegrationEvent` | `Billing.IssueInvoiceIntegrationCommand` | `__TypeId__` |
| `id` | UUIDv7 da linha | UUIDv7 da linha | `idempotency-key` |
| `correlation_id` | vazio | id que liga comando e resposta | `kafka_correlationId` |
| `reply_topic` | vazio | `ordering.saga.replies` | `kafka_replyTopic` |

Os dois headers novos são exatamente os do `CommandHeaders` do [contrato](#o-contrato-da-orquestração), com os nomes que o Spring Kafka usa (`KafkaHeaders.CORRELATION_ID` e `KafkaHeaders.REPLY_TOPIC`). O `OutboxKafkaSender` só os adiciona quando a coluna está preenchida, então um evento sai exatamente como antes.

### `event_type` → `message_type`

A V011 só renomeia uma coluna, e o motivo é de vocabulário: a coluna passa a guardar o tipo de **comandos** também, e `event_type` começaria a mentir. O mesmo rename vale para o campo `OutboxMessage.messageType` e para o método `readMessageType` do recorder. Linhas pendentes continuam válidas, porque só o nome mudou. Como toda migração de schema, ela deve rodar com o serviço parado.

### `OutboxDraft`: o rascunho da mensagem

```java
@Builder
public record OutboxDraft(String channelName, String aggregateId,
                          String correlationId, String replyTopic,
                          Object payload) { }
```

Antes, `OutboxRecorder.record(channelName, aggregateId, message)` tinha três parâmetros posicionais. Com comandos seriam cinco, dois deles opcionais: uma assinatura em que é fácil trocar dois `String` de lugar sem o compilador perceber. O `OutboxDraft` é um **parâmetro-objeto** com builder: cada campo vai pelo nome, e o que não se aplica simplesmente não é preenchido.

A divisão de responsabilidades ficou explícita:

- **O publisher diz o quê e para onde**: canal, agregado, payload e, se for comando, correlação e tópico de resposta.
- **O recorder decide o resto**: gera o id da linha (UUIDv7, que também é a idempotency key), serializa o payload com o mesmo `JacksonJsonSerializer` do caminho Kafka, extrai o `__TypeId__` lógico dos headers da serialização e inicializa o estado de envio (tentativas, próximo envio). Continua `@Transactional(propagation = MANDATORY)`: gravar fora da transação de negócio é recusado.

```mermaid
flowchart LR
    A["publisher<br/>(evento ou comando)"] -->|"OutboxDraft"| B["OutboxRecorder.record<br/>MANDATORY"]
    B --> C[("outbox<br/>message_type · correlation_id · reply_topic")]
    C -.->|"OutboxDispatcher (agendado)"| D["OutboxKafkaSender"]
    D -->|"key = aggregate_id<br/>__TypeId__, idempotency-key<br/>+ kafka_correlationId, kafka_replyTopic se houver"| E[("tópico")]
```

O `OutboxOrderIntegrationEventPublisher` foi o primeiro a usar o draft: monta um `OutboxDraft` só com canal, agregado e payload, sem correlação nem resposta.

### Revisão desta etapa

- **Corrigido**: o comentário do `OutboxKafkaSender` ainda dizia `__TypeId__ = eventType` e não citava os headers novos. O `OutboxDraft` ganhou comentário explicando o papel do rascunho.
- **Ainda não há publisher de comando**: a outbox está pronta para comandos, mas nenhum código grava um com `correlationId`/`replyTopic`. Isso vem com o envio do `IssueInvoice`.
- **Testes**: `./gradlew check` do ordering verde (304 unitários, 12 de contrato, 185 de integração). O `CheckoutApplicationServiceIT` grava o `OrderPlaced` na outbox, o que exercita a gravação com as colunas da V011/V012. O envio com os headers novos não tem teste: um IT do `OutboxKafkaSender` com correlação e resposta preenchidas é o que travaria esse caminho.

---

## O primeiro comando: emitir a fatura

Até aqui a saga nascia e parava em `PLACING_ORDER`. Agora ela dá o primeiro passo do [mapa](#o-fluxo-passo-a-passo): manda o comando `Issue Invoice` ao billing e passa a esperar a resposta em `INVOICING`.

```mermaid
sequenceDiagram
    participant CK as checkout v1 (@Transactional)
    participant SC as PlaceOrderSagaCoordinator.start
    participant S as PlaceOrderSaga
    participant P as OutboxPlaceOrderSagaIntegrationCommandPublisher
    participant DB as Postgres (mesma transação)
    CK->>SC: OrderPlacedEvent → start(orderId)
    SC->>DB: orders.ofId → OrderSnapshot
    Note over SC: IssueInvoiceIntegrationCommand.from(snapshot)
    SC->>S: start(orderId) + moveToInvoicing()
    SC->>DB: sagas.add → saga_instance (RUNNING, INVOICING)
    SC->>P: send(sagaId, command)
    P->>DB: OutboxDraft → outbox<br/>canal billing.invoice.commands, correlation_id = sagaId,<br/>reply_topic = ordering.saga.replies
    Note over DB: commit: pedido + OrderPlaced + saga + comando
    DB-->>DB: depois, o OutboxDispatcher envia o comando ao Kafka
```

### As peças, por camada

| Camada | Peça | O que faz |
|---|---|---|
| aplicação | `IssueInvoiceIntegrationCommand` | O comando: pedido congelado (itens, frete, cobrança, `PaymentSnapshot` com método e cartão). Implementa `OutboundIntegrationCommand`, key = `orderId` |
| aplicação | `PlaceOrderSaga.moveToInvoicing()` | Primeira transição: `require(isRunningAt(PLACING_ORDER))` e `changeStepTo(INVOICING)`. O status continua `RUNNING` |
| aplicação | `PlaceOrderSagaCoordinator.start` | Carrega o pedido, monta o comando, inicia e avança a saga, grava, envia |
| porta de saída | `ForPublishingPlaceOrderSagaIntegrationCommands.send(correlationId, command)` | O coordenador diz *qual* comando e de *qual* saga; o resto é do adaptador |
| adaptador | `OutboxPlaceOrderSagaIntegrationCommandPublisher` | Monta o `OutboxDraft` com canal, agregado, `correlationId` e `replyTopic` e grava na outbox |
| adaptador | `PlaceOrderSagaCommandTopics.of(command)` | Roteia cada comando para o tópico do participante, por pattern matching de tipo |
| config | `AlgaShopMessagingKafkaProperties` + `KafkaConfig` (ordering) | `billing-invoice-commands-topic-name`, `saga-replies-topic-name`; cria `ordering.saga.replies` e seu DLT |
| config | `AlgaShopMessagingKafkaProperties` + `KafkaConfig` (billing) | `invoice-commands-topic-name`, `invoice-commands-consumer-group`; cria `billing.invoice.commands` |

### Três decisões para guardar

**1. O comando reaproveita o snapshot.** O `toSnapshot(Order)` que [ficou sem chamador](#o-ordersnapshot-encolhe) quando o checkout assíncrono saiu volta aqui, como estava previsto. O comando carrega o pedido inteiro (ECST): o billing emite a fatura sem chamar o ordering de volta. Os itens são ordenados por id, para o mesmo pedido gerar sempre o mesmo JSON.

**2. `correlationId = sagaId`.** A resposta do billing vai voltar com o mesmo `kafka_correlationId`, e o orquestrador vai usá-lo para achar a saga, checar com `isRunningAt(INVOICING)` se ainda espera aquela resposta, e avançar. O `orderId` também identificaria o pedido, mas o `sagaId` identifica a **execução**: se um dia houver mais de uma saga por pedido (outro tipo, ou uma nova tentativa), a correlação continua sem ambiguidade.

**3. Quem cria cada tópico é quem o consome.** O billing executa os comandos de fatura, então ele declara `billing.invoice.commands`. O ordering consome as respostas, então ele declara `ordering.saga.replies` e o DLT dela. O ordering só conhece o *nome* do tópico de comandos, pela property, e não o cria. É a mesma regra dos tópicos de eventos (cada dono declara o seu), aplicada ao lado de quem recebe.

### O roteamento de tópico

```java
return switch (command) {
    case IssueInvoiceIntegrationCommand ignored -> properties.getBillingInvoiceCommandsTopicName();
    default -> throw new IllegalArgumentException("Unsupported saga command " + ...);
};
```

O publisher não precisa saber para onde vai cada comando: pergunta ao `PlaceOrderSagaCommandTopics`. Cada comando novo (`ReserveStock`, `CancelInvoice`) ganha uma linha no `switch`. Um comando sem rota **falha alto** no envio: melhor uma exceção (que desfaz a transação inteira) do que um comando gravado para um tópico que ninguém consome.

### Revisão desta etapa

- ~~**Corrigido**: o alias `Billing.IssueInvoiceIntegrationCommand` foi trocado por `Ordering.`~~ **Revertido**: `Billing.` estava certo. Os dois lados (ordering e billing) usam `Billing.IssueInvoiceIntegrationCommand`, e o contrato foi alinhado a eles. Ver [a convenção dos `__TypeId__`](#a-convenção-dos-__typeid__).
- **Corrigido**: os `application-test-env.yaml` do ordering e do billing não tinham as properties novas (`@NotBlank`). O contexto dos testes de integração não subiria. Adicionadas, e o ordering ganhou também o alias do comando.
- **Comentários**: cada classe nova ou alterada ganhou um comentário curto explicando seu papel.
- **Testes**: ordering `check` verde (304 unitários, 12 de contrato, 185 de integração); billing `check` verde (48 unitários, 20 de integração). O `CheckoutApplicationServiceIT` agora executa o caminho inteiro: grava pedido, saga em `INVOICING` e o comando na outbox.
- **Pendências**:
  - o billing ainda não **consome** o comando, não tem listener nem DLT para `billing.invoice.commands`, e não responde. O comando fica no tópico até a próxima aula;
  - a pendência da [persistência da saga](#revisão-desta-etapa-1) continua (atualizar uma saga já gravada perde `createdAt` e a `version`). Ainda não morde, porque a saga só é inserida, mas a primeira resposta vai exigir um `update`;
  - o comando não passa por Bean Validation antes de ir para a outbox (o `OutboxRecorder` não valida). As anotações do `IssueInvoiceIntegrationCommand` só protegem o lado de quem consome.

---

## O billing executa o comando

O comando `IssueInvoice` que o ordering grava na outbox agora tem quem o consuma. Do lado do billing, ele vira uma fatura pelo mesmo caso de uso que sempre existiu: o `generate`.

```mermaid
flowchart LR
    A[("billing.invoice.commands<br/>key = orderId<br/>kafka_correlationId · kafka_replyTopic")] -->|"grupo billing.invoice-commands"| B["KafkaInvoiceIntegrationCommandListener"]
    B -->|"headers presentes?"| C{"sim"}
    B -->|"faltou header"| X["IllegalArgumentException<br/>→ DLT"]
    C --> D["@Valid IssueInvoiceIntegrationCommand"]
    D --> E["GenerateInvoiceInputAssembler"]
    E --> F["InvoiceManagementApplicationService.generate<br/>@Transactional"]
    F --> G[("invoice<br/>UNIQUE (order_id)")]
    F -.->|"after commit"| H[("billing.invoice.events<br/>InvoiceIssued")]
```

### As peças

| Peça | O que é |
|---|---|
| `InboundIntegrationCommand` | Marcador dos comandos que o billing **recebe**. Simétrico ao `OutboundIntegrationCommand` do ordering: quem recebe não precisa de key |
| `invoice/command/IssueInvoiceIntegrationCommand` | Cópia local do contrato, com Bean Validation. É o mesmo pedido congelado que o ordering montou |
| `application/order/snapshot/*` (seis records) | Voltaram: tinham saído junto com o `OrderPlacedIntegrationEvent` na remoção da coreografia, e agora chegam dentro do comando |
| `GenerateInvoiceInputAssembler` | Voltou, agora a partir do comando. A tradução é a mesma de antes: itens viram linhas, frete maior que zero vira a linha "Shipping", o payer vem do `BillingSnapshot` |
| `KafkaInvoiceIntegrationCommandListener` | Consome `billing.invoice.commands` no grupo `billing.invoice-commands`, valida e chama `generate` |
| `KafkaConfig` | Ganhou o DLT `billing.dlt.billing.invoice.commands` |

A peça que **não** mudou é a mais importante: o `InvoiceManagementApplicationService.generate`. Ele ficou preservado na remoção da coreografia exatamente para isto. Na coreografia, quem o chamava era um evento (`OrderPlaced`); agora é um comando. O caso de uso não sabe a diferença.

### Comando desconhecido é erro; evento desconhecido, não

O handler default do listener de comandos **lança** exceção:

```java
@KafkaHandler(isDefault = true)
public void handle(@Payload Object command, ...) {
    throw new IllegalArgumentException("Unsupported invoice command type " + ...);
}
```

Os listeners de **evento** do projeto fazem o contrário: logam e ignoram. A diferença é de natureza:

- Um **evento** novo é evolução normal do produtor. O consumidor que não se interessa por ele ignora, e nada está errado.
- Um **comando** é uma ordem endereçada a este serviço. Se ele chegou e o billing não sabe executá-lo, o contrato quebrou. Ignorar em silêncio deixaria o orquestrador esperando uma resposta que nunca vem. Lançar manda a mensagem para o DLT, onde alguém vai vê-la.

### Os headers são obrigatórios

Antes de executar, o listener confere `kafka_correlationId` e `kafka_replyTopic`. Sem eles não há como responder ao orquestrador, então a falta é tratada como mensagem inválida (`IllegalArgumentException`, não-retryable, DLT). Hoje o listener só **valida** os headers; a resposta (`InvoicePaidReply`/`InvoiceCanceledReply` no tópico do `kafka_replyTopic`) vem na próxima etapa.

### A convenção dos `__TypeId__`

Esta etapa fixou a regra que estava ambígua no contrato:

| Tipo de mensagem | Prefixo | Exemplos |
|---|---|---|
| **Evento** | serviço que **publica** o fato | `Ordering.OrderPlacedIntegrationEvent`, `Billing.InvoicePaidIntegrationEvent` |
| **Comando** | serviço que **executa** a ordem | `Billing.IssueInvoiceIntegrationCommand`, `Billing.CancelInvoiceIntegrationCommand`, `ProductCatalog.ReserveStockIntegrationCommand` |
| **Resposta** | serviço que **responde** | `Billing.InvoicePaidIntegrationReply`, `ProductCatalog.StockReservationConfirmedIntegrationReply` |

O fato pertence a quem o produziu; o comando pertence ao vocabulário de quem o executa. `IssueInvoice` é uma operação do billing, e o ordering apenas a solicita. Ordering e billing já usavam `Billing.IssueInvoiceIntegrationCommand`; o contrato (`messages.yml`) foi alinhado a eles, e as duas "correções" para `Ordering.` feitas nas etapas anteriores deste doc foram revertidas. O YAML de teste do ordering, que tinha ficado com `Ordering.`, também foi corrigido, e o do billing ganhou o alias.

### Revisão desta etapa

- **Testes**: billing `check` verde (48 unitários, 20 de integração); ordering `check` verde (304, 12, 185). Nos testes do billing o listener não sobe (`auto-startup: false`), então o consumo do comando ainda não tem teste próprio. Um IT que envie o comando e verifique a fatura gerada fecharia esse buraco.
- **Comentários**: o listener, o `InboundIntegrationCommand` e o assembler ganharam comentários curtos.
- **Pendências**:
  - **duplicata vira DLT de novo**: um `IssueInvoice` reentregue bate no `UNIQUE (order_id)`, lança `DataIntegrityViolationException` (não-retryable) e vai para o DLT. Na coreografia isso só poluía o DLT. Na orquestração é pior: se a primeira execução já gerou a fatura e a resposta se perdeu, a reentrega deveria **responder de novo**, e não falhar. Quando a resposta existir, o caso "fatura já existe para este pedido" precisa reenviar o reply;
  - o `generate` publica `InvoiceIssued` depois do commit, e o próprio billing ainda consome esse evento para cobrar (o Listen to Yourself da aula 7.8). É a cobrança, por esse caminho, que vai produzir o `InvoicePaid`/`InvoiceCanceled` que a resposta precisa;
  - a resposta ao orquestrador ainda não existe.

---

## Próximas aulas (a documentar)

O modelo, a persistência, o início da saga e o contrato já existem (seções acima). Os métodos de transição, os comandos e as respostas em código entram aqui conforme o módulo avança. A comparação com a coreografia vai seguir as perguntas que o [balanço da 8.13](./saga-coreografada.md#813--opinião-sobre-saga-com-coreografia) deixou: onde fica o estado "em que passo está o pedido 42"? Quem decide a compensação? O que acontece com os três kits de consumidor copiados?
