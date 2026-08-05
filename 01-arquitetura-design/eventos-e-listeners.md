# Eventos, Listeners e Consistência Eventual

> Como o `product-catalog` passou a anunciar fatos em vez de só gravar estado — e como a cópia da categoria dentro de cada produto se mantém em dia sem travar quem chamou a API.
> Código real: `domain/product/Product.java`, `domain/product/StockService.java`, `application/ApplicationMessagePublisher.java`, `domain/DomainEventPublisher.java`, `infrastructure/listener/`, `infrastructure/persistence/category/ProductCategoryUpdater.java`.

> Este documento é a continuação de [`desnormalizacao-mongo.md`](../02-persistencia/desnormalizacao-mongo.md). Lá se decidiu **copiar** o nome da categoria para dentro do produto; aqui se paga a conta dessa decisão.

---

## O problema

Desnormalizar resolveu a leitura e criou um problema novo, e ele é inevitável: **cópia envelhece**.

```
PUT /api/v1/categories/{id}   →   { "name": "Notebooks" }

coleção categories:  name = "Notebooks"     ← atualizado
coleção products:    category.name = "Laptops"  ← 12 documentos mentindo
```

Alguém precisa reescrever esses doze produtos. As opções, e por que a escolhida ganhou:

| Opção | Por que não / por que sim |
|---|---|
| O `CategoryManagementApplicationService` chama o updater direto | ❌ acopla gestão de categoria a conhecimento do catálogo de produtos, e faz quem chamou o `PUT` esperar a reescrita de N documentos |
| Um job periódico varrendo e reconciliando | ❌ a janela de inconsistência passa a ser o intervalo do job, não o tempo de uma escrita |
| Nada — aceitar dado velho | ❌ o nome exibido nunca convergiria |
| **Publicar um evento e reagir a ele** | ✅ quem atualiza categoria não precisa saber quem se interessa; o consumidor pode ser assíncrono e trocado sem tocar no publicador |

---

## Três mecanismos diferentes no mesmo serviço

O serviço tem **três** famílias de evento, e elas não são a mesma coisa com nomes diferentes:

| | Eventos do agregado (`Product`) | Evento de aplicação (`CategoryUpdatedEvent`) | Eventos de serviço de domínio (estoque) |
|---|---|---|---|
| Quem registra | o próprio agregado | o application service, explicitamente | o `StockService`, explicitamente |
| Quando é publicado | no `productRepository.save()` | na chamada a `send()` | na chamada a `publish()` |
| Quem publica | `EventPublishingRepositoryProxyPostProcessor` (Spring Data) | `ApplicationMessagePublisher` | `DomainEventPublisher` |
| Onde mora a classe | `domain/product/` | `application/category/event/` | `domain/product/` |
| Listener | `ProductEventListener`, **síncrono** (exceto `PriceChanged`) | `CategoryEventListener`, **`@Async`** | `ProductEventListener`, **síncrono** |
| Se o listener falhar | a exceção sobe para quem salvou | vira log, e a cópia fica velha | **o rollback desfaz o ajuste de estoque** |

A distinção entre os dois primeiros: **um evento de domínio é um fato que o agregado descobriu sozinho** ("este produto entrou em promoção"). Um evento de aplicação é uma **notificação que alguém decidiu emitir** porque outro pedaço do sistema tem interesse. Fazer a `Category` emitir `CategoryUpdatedEvent` seria dar a ela conhecimento de um problema que é do vizinho — quem tem cópia de categoria é o produto, e a categoria não precisa saber disso.

O terceiro é de domínio como o primeiro — `ProductSoldOutEvent` é um fato sobre o produto — mas **não pode usar o mesmo caminho**, e a razão é puramente técnica: o ajuste de estoque não carrega nem salva o agregado, então não existe `save()` para disparar a publicação. Ver [`concorrencia-e-atomicidade.md`](../02-persistencia/concorrencia-e-atomicidade.md).

> #### Nota de estudo: duas portas quase idênticas, de propósito
>
> `ApplicationMessagePublisher` (na `application`) e `DomainEventPublisher` (no `domain`) têm a mesma assinatura e são implementadas pelo **mesmo** `ApplicationEventPublisher` do Spring, em dois `@Configuration` separados. Parece duplicação, e em parte é — mas o que as separa é **camada**, não comportamento: o domínio não pode depender de uma interface declarada na `application`, ou a seta de dependência aponta para o lado errado.
>
> O custo está registrado: duas interfaces gêmeas e dois beans para manter, e um leitor desavisado escolhe a errada com facilidade. A alternativa — uma porta só, no domínio — economizaria código ao preço de fazer o `CategoryUpdatedEvent`, que não é evento de domínio, viajar por uma porta do domínio. Foi essa a troca escolhida; vale conhecer as duas pontas.

---

## Eventos de domínio — `AbstractAggregateRoot`

```java
// domain/product/Product.java
public class Product extends AbstractAggregateRoot<Product> {

    public void changePrice(BigDecimal regularPrice, BigDecimal salePrice) {
        // ... valida e aplica ...
        if (pricesDidNotChange(oldRegularPrice, oldSalePrice)) {
            return;
        }
        registerPriceChangedEvent(oldRegularPrice, oldSalePrice);
        if (isNewlyOnSale(wasOnSale)) {
            registerProductPlacedOnSaleEvent();
        }
    }
}
```

A superclasse dá dois membros: `registerEvent()` e uma lista `@Transient` de eventos pendentes — transiente, então ela **não vai para o documento**.

### ⚠️ `registerEvent` não publica nada

Este é o ponto que mais surpreende: `registerEvent()` só **enfileira**. Quem publica é o `EventPublishingRepositoryProxyPostProcessor` do Spring Data, um interceptador que age **depois** de um `save()` feito por um repositório — e que limpa a fila em seguida.

```
product.changePrice(...)          → evento enfileirado, nada acontece
productRepository.save(product)   → grava, DEPOIS publica, DEPOIS limpa a fila
```

As consequências práticas:

| Situação | Publica? |
|---|---|
| `productRepository.save(product)` | ✅ sim |
| `mongoOperations.updateMulti(...)` (o `ProductCategoryUpdater`) | ❌ não — não passa por repositório |
| Agregado modificado e nunca salvo | ❌ não |
| Produto **lido** do banco | ❌ não — a materialização usa o construtor protegido, que não registra nada |

Essa última linha é o motivo de `registerProductAddedEvent()` poder morar dentro do construtor público sem inundar o sistema: o Spring Data instancia por outro caminho.

### Os cinco eventos, e por que são cinco

| Evento | Quando | Por que existe separado |
|---|---|---|
| `ProductAddedEvent` | no construtor | o catálogo ganhou um item |
| `ProductPriceChangedEvent` | em `changePrice`, se mudou | leva o preço **antigo e o novo** — a diferença é o que interessa, e o documento só guarda o agora |
| `ProductPlacedOnSaleEvent` | junto do anterior, só quando o desconto **nasce** | quem dispara notificação não quer ser acordado por toda alteração de preço |
| `ProductListedEvent` | em `setEnabled`, ao voltar para a vitrine | par do próximo |
| `ProductDelistedEvent` | em `setEnabled`, ao sair da vitrine | "delisted", não "deleted" — o documento continua lá |

Dois eventos para uma única mudança de preço não é redundância: são fatos de níveis diferentes. *"O preço mudou"* interessa a auditoria; *"entrou em promoção"* interessa a quem notifica cliente.

E o agregado é cuidadoso com o que chama de fato:

```java
// nada mudou, então não houve fato — sai sem registrar
if (pricesDidNotChange(oldRegularPrice, oldSalePrice)) {
    return;
}
```

```java
// setEnabled: só emite quando a situação MUDA
if (wasEnabled != null && wasEnabled && !this.getEnabled()) {
    registerDelistedProductEvent();
} else if (wasEnabled != null && !wasEnabled && this.getEnabled()) {
    registerListedProductEvent();
}
```

A guarda `wasEnabled != null` distingue *"produto sendo criado"* de *"produto sendo alterado"*. Sem ela, todo produto nascido com `enabled = true` emitiria um `Listed` logo após o `ProductAddedEvent`, anunciando que foi listado algo que acabou de existir. E comparar antes/depois evita evento em chamada idempotente: `disable()` num produto já inativo não é um fato.

Está tudo coberto em `src/test/.../domain/product/ProductTest.java`, que inspeciona a fila sem subir Spring nenhum.

---

## Evento de aplicação — a porta de saída

A aplicação declara **o que precisa**, sem dizer como:

```java
// application/ApplicationMessagePublisher.java
public interface ApplicationMessagePublisher {
    void send(Object message);
}
```

E a infraestrutura fornece o como:

```java
// infrastructure/message/ApplicationMessagePublisherConfig.java
@Bean
public ApplicationMessagePublisher applicationMessagePublisher(
        ApplicationEventPublisher applicationEventPublisher) {
    return applicationEventPublisher::publishEvent;
}
```

Uma *method reference* basta — a interface tem um método só. É [porta e adaptador](./ports-hexagonal.md) no formato mais enxuto possível, e o ganho é concreto: o `CategoryManagementApplicationService` não importa nada de Spring para publicar, e o dia em que isso virar RabbitMQ ou Kafka, **quem muda é o `@Bean`** — nenhum service é tocado.

O publicador:

```java
// application/category/management/CategoryManagementApplicationService.java
public void update(UUID categoryId, CategoryInput input) {
    Category category = categoryRepository.findById(categoryId)
            .orElseThrow(() -> new CategoryNotFoundException(categoryId));
    category.setName(input.getName());
    category.setEnabled(input.getEnabled());
    categoryRepository.save(category);

    applicationMessagePublisher.send(new CategoryUpdatedEvent(
            category.getId(), category.getName(), category.getEnabled()));
}
```

Três detalhes deliberados:

1. **Publica depois de gravar.** O evento afirma um fato consumado; consumidor que reagisse a uma gravação que ainda pode falhar propagaria mentira.
2. **`disable()` publica o mesmo evento.** Desabilitar também muda `category.enabled`, e a cópia precisa refletir.
3. **`create()` não publica.** Categoria recém-criada não tem produto apontando para ela — não há cópia a sincronizar.

O evento leva `name` e `enabled`, não só o id, para o consumidor não precisar reler a categoria que acabou de ser gravada. O payload é exatamente o que o `ProductCategory` copia: crescer um sem crescer o outro não faria sentido.

---

## Evento de domínio sem `save()` — a terceira porta

Os cinco eventos acima dependem de uma condição que parecia universal: **alguém chama `productRepository.save()`**. A Fase 13 quebrou essa premissa.

O ajuste de estoque não carrega o agregado. Ele altera o documento direto, com uma operação condicional atômica, porque é a única forma de dois saques simultâneos não se atropelarem — ver [`concorrencia-e-atomicidade.md`](../02-persistencia/concorrencia-e-atomicidade.md). Sem `save()`, o `EventPublishingRepositoryProxyPostProcessor` nunca é acionado, e um `registerEvent()` no `Product` ficaria enfileirado para sempre.

Daí a porta:

```java
// domain/DomainEventPublisher.java
public interface DomainEventPublisher {
    void publish(Object event);
}
```

```java
// domain/product/StockService.java — serviço de domínio
if (result.isOutOfStock()) {
    domainEventPublisher.publish(
            ProductSoldOutEvent.builder().productId(product.getId()).build());
}
```

`ProductSoldOutEvent` e `ProductRestockedEvent` são eventos de domínio como os outros cinco — o que muda é **quem os publica**, e a razão é técnica, não conceitual.

### Transição, não estado

Os dois só saem quando a quantidade **cruza** o zero:

| Antes | Depois | Evento |
|---|---|---|
| 50 | 40 | — |
| 10 | 0 | `ProductSoldOutEvent` |
| 0 | 0 | — |
| 0 | 10 | `ProductRestockedEvent` |
| 40 | 50 | — |

É o mesmo cuidado da guarda `wasEnabled != null` do `setEnabled`: sem a condição sobre o valor anterior, "está zerado" seria verdade a cada operação e viraria uma enxurrada de eventos idênticos. É o que separa um **fato** de uma **leitura de estado**.

> 🔧 **Isto mudou na Fase 14.** Até então este era o único dos três mecanismos que publicava sem rede nenhuma: o `publish()` acontecia depois de o estoque já ter mudado, e uma falha ali deixava o estoque baixado sem que ninguém fosse avisado.
>
> Hoje `restock` e `withdraw` são `@Transactional`, e estes listeners são **síncronos** — então eles rodam **dentro** do commit. Uma exceção no listener desfaz o ajuste de estoque junto: o evento deixou de ser um "depois" e virou parte da mesma operação.
>
> A garantia, porém, está a **uma anotação** de distância do fim. Um `@Async` em qualquer um destes handlers o tiraria do rollback — outra thread significa fora da sessão transacional —, e ele passaria a ler o estado anterior ao commit. Nada quebraria, nada avisaria. Ver [`transacoes-mongo.md`](../02-persistencia/transacoes-mongo.md).

---

## O consumidor assíncrono

```java
// infrastructure/listener/category/CategoryEventListener.java
@EventListener
@Async
public void handle(CategoryUpdatedEvent categoryUpdatedEvent) {
    productCategoryUpdater.copyCategoryDataToProducts(categoryUpdatedEvent);
    log.info("Category updated received: {}", categoryUpdatedEvent.getCategoryId());
}
```

```java
// infrastructure/persistence/category/ProductCategoryUpdater.java
Query query = new Query(Criteria.where("category._id").is(event.getCategoryId()));

Update update = new Update()
        .set("category.name", event.getName())
        .set("category.enabled", event.getEnabled());

mongoOperations.updateMulti(query, update, Product.class);
```

`updateMulti` pede ao Mongo que percorra e altere no servidor, em vez de carregar N produtos para a JVM, chamar `setCategory` em cada um e salvar de volta.

> ⚠️ **Desde a Fase 15, esta propagação tem um consumidor que ela não alcança: o cache.** O `updateMulti` reescreve a cópia da categoria dentro dos produtos **no Mongo** — e nada invalida as entradas de `algashop:products:v1` no Redis. Um produto em cache continua servindo o nome antigo da categoria até o TTL de 1 minuto expirar.
>
> O detalhe que torna isso fácil de não ver: o `@CacheEvict` do `CategoryManagementApplicationService` funciona e limpa os caches **de categoria**. O que fica velho é o cache de **produto**, atualizado por um caminho completamente diferente — assíncrono, em outra thread, sem passar por nenhum método anotado.
>
> A resposta escolhida foi encurtar o TTL em vez de acoplar o listener ao cache. A troca inteira está em [`cache.md`](./cache.md#o-que-não-existe-invalidação-por-evento).

O `@Async` depende de `@EnableAsync`, que mora em `infrastructure/async/AsyncConfig.java`. **Sem essa anotação o `@Async` é silenciosamente ignorado** — o método roda na mesma thread, sem nenhum erro, e a única pista seria a propagação acontecer síncrona.

> #### Nota de estudo: `@Async` deixou de ser uma decisão local
>
> Na Fase 14 o handler de `ProductPriceChangedEvent` — no `ProductEventListener`, não neste — também ganhou `@Async`. Hoje é inofensivo: `changePrice` sai pelo `update()`, que não é transacional.
>
> Mas a régua mudou. Enquanto nenhum fluxo tinha transação, `@Async` era uma escolha sobre **latência**: quem espera o quê. Com `@Transactional` em cena, ele passou a ser também uma escolha sobre **atomicidade**, porque a thread nova não enxerga a sessão transacional. A mesma anotação, no mesmo arquivo, significa duas coisas diferentes dependendo de o método que publicou o evento ser transacional ou não — e nada no código diz qual dos dois casos você está lendo.

---

## ⚠️ O que a consistência eventual custa aqui

O `@Async` é uma troca deliberada: quem chama `PUT /categories/{id}` recebe a resposta assim que a categoria é gravada, sem esperar a reescrita dos produtos. Em troca, existe uma janela em que a categoria já tem o nome novo e a listagem ainda mostra o antigo.

Isso é exatamente o **E** de BASE — *eventually consistent* — descrito em [`nosql-conceitos.md`](../02-persistencia/nosql-conceitos.md): *"aceito inconsistência temporária em troca de uptime e escala"*. Aqui a teoria daquele documento virou linha de código.

O que **não existe** nesta implementação, e convém enxergar com clareza:

| Falta | Consequência |
|---|---|
| Retentativa | `updateMulti` que estoure não é tentado de novo |
| Fila persistente | o `ApplicationEventPublisher` é in-process: a "mensagem" nunca sai da JVM e **não sobrevive a uma queda** |
| Dead letter | evento perdido não vai para lugar nenhum — some |
| Ordem garantida | dois updates seguidos da mesma categoria podem ser aplicados fora de ordem |
| Tratamento de erro | a exceção morre na thread do executor; sem handler, nem log aparece |
| Reconciliação | nada compara depois e corrige o que ficou para trás |

> ⚠️ **Falha silenciosa é a pior delas.** Se o `updateMulti` lançar, os produtos ficam com o nome velho **para sempre**, e ninguém é avisado. Repare que o `log.info` do listener vem **depois** da chamada ao updater — se ela estourar, nem a linha de log sai. Com broker de verdade isso viraria retry e dead letter; hoje é uma pendência declarada.

Vale também notar: `@EventListener` e não `@TransactionalEventListener`, porque não há transação envolvida — o service já gravou a categoria antes de publicar.

E, porque a escrita vai por `MongoOperations` direto, o `@Version` do produto **não** é incrementado: a propagação escapa do lock otimista.

---

## O executor

Não há executor declarado — fica o que o Spring Boot autoconfigura (`applicationTaskExecutor`). O default tem **fila ilimitada**, então uma rajada de updates de categoria enfileira em memória em vez de rejeitar. Em ambiente de estudo isso não aparece; com carga real, dimensionar deixa de ser opcional.

---

## Ver funcionando

O jeito mais direto de entender consistência eventual é provocá-la:

```bash
curl -s "localhost:8083/api/v1/products?size=1" | jq '.content[0].category'
# { "id": "e0c4271d-...", "name": "Laptops", "enabled": true, "slug": "laptops" }

curl -sX PUT localhost:8083/api/v1/categories/e0c4271d-0016-4a42-82fe-bf695a9fb9b8 \
     -H 'Content-Type: application/json' -d '{"name":"Notebooks","enabled":true}'

# no log da aplicação:
#   Category updated received: e0c4271d-0016-4a42-82fe-bf695a9fb9b8

curl -s "localhost:8083/api/v1/products?size=1" | jq '.content[0].category'
# { "id": "e0c4271d-...", "name": "Notebooks", "enabled": true, "slug": "notebooks" }
```

Os eventos de domínio aparecem no mesmo log, ao salvar um produto:

```
ProductAddedEvent: ProductAddedEvent(productId=..., addedAt=...)
ProductPriceChangedEvent: ProductPriceChangedEvent(productId=..., oldSalePrice=2789, newSalePrice=2500, ...)
ProductPlacedOnSaleEvent: ProductPlacedOnSaleEvent(productId=..., ...)
```

---

## Armadilhas

1. **`registerEvent` não publica.** Só o `save()` por repositório publica. Escrita por `MongoTemplate`/`MongoOperations` é invisível para o mecanismo — foi exatamente por isso que o estoque precisou da terceira porta.
2. **`@Async` sem `@EnableAsync` é ignorado em silêncio.** Roda síncrono, sem erro nenhum.
3. **Exceção em `@Async` some.** Sem `AsyncUncaughtExceptionHandler`, um método `void` assíncrono engole a falha.
4. **`@EventListener` não é transacional.** Se um dia houver transação, o listener rodará dentro dela e um rollback publicaria fato que não aconteceu — aí o certo passa a ser `@TransactionalEventListener(phase = AFTER_COMMIT)`.
5. **Evento in-process não sobrevive a restart.** Derrubar a aplicação entre o `send()` e o `updateMulti` perde a propagação sem deixar rastro.
6. **Ordem entre eventos não é garantida.** Dois renames seguidos podem chegar invertidos, e o último a executar vence — não o último a ser publicado.
7. **Publicar antes de gravar propaga mentira.** A ordem `save()` → `send()` não é estilo, é correção.

---

## Pendências registradas

- [ ] **Não há retentativa nem dead letter.** Falha na propagação é definitiva e silenciosa. Enquanto não houver broker, um `try/catch` com `log.error` no listener já seria melhor que o silêncio de hoje.
- [ ] **Não há reconciliação.** Nenhuma rotina compara `category.name` dos produtos com a coleção `categories` para corrigir o que escapou.
- [ ] **O executor do `@Async` é o default, com fila ilimitada.** Sem dimensionamento nem métrica.
- [ ] **A propagação escapa do `@Version`.** Escrita por `MongoOperations` não incrementa a versão do produto.
- [ ] **`updateMulti` não usa índice.** Ver [`desnormalizacao-mongo.md`](../02-persistencia/desnormalizacao-mongo.md).
- [ ] **Os eventos de domínio não têm consumidor de verdade.** `ProductEventListener` só registra em log — o que é proposital nesta etapa, para tornar visível *quando* cada evento sai.
- [ ] **Duas portas gêmeas de publicação.** `ApplicationMessagePublisher` e `DomainEventPublisher` têm a mesma assinatura e o mesmo bean por trás; a separação é de camada, e o custo é escolher a errada sem perceber.
- [ ] **Os eventos de estoque publicam sem rede nenhuma.** Sem transação e sem `@Async`: falha na publicação significa estoque alterado e ninguém avisado, sem reparo posterior.
- [ ] **Mensageria entre serviços continua não existindo.** Os eventos são internos ao processo, aqui e no `ordering`. Ver [`arquitetura.md`](../00-visao-geral/arquitetura.md).

---

## Checklist de revisão

- [ ] Sei distinguir evento de domínio de evento de aplicação, e por que cada um mora onde mora
- [ ] Sei que `registerEvent` só enfileira, e que quem publica é o `save()` do repositório
- [ ] Sei por que ler um produto do banco não dispara `ProductAddedEvent`
- [ ] Entendo por que `changePrice` pode emitir dois eventos, e por que pode não emitir nenhum
- [ ] Sei explicar a guarda `wasEnabled != null` do `setEnabled`
- [ ] Entendo o que a `ApplicationMessagePublisher` isola, e o que mudaria ao trocar por um broker
- [ ] Sei por que o evento é publicado depois do `save()`, e nunca antes
- [ ] Sei o que o `@Async` compra e o que ele cobra
- [ ] Sei apontar, na lista, tudo que falta para isso virar mensageria de verdade
- [ ] Consigo provocar a janela de inconsistência e observá-la fechando
- [ ] Sei por que os eventos de estoque não podem usar o `AbstractAggregateRoot`
- [ ] Sei por que existem duas portas de publicação, e o que se ganharia e se perderia unificando

---

## Referências

- [Spring Data — Publishing Events from Aggregate Roots](https://docs.spring.io/spring-data/commons/reference/repositories/core-domain-events.html)
- [Spring Framework — Application Events](https://docs.spring.io/spring-framework/reference/core/beans/context-introduction.html#context-functionality-events)
- [Spring Framework — Asynchronous Execution (`@Async`)](https://docs.spring.io/spring-framework/reference/integration/scheduling.html#scheduling-annotation-support-async)
- [Martin Fowler — Domain Event](https://martinfowler.com/eaaDev/DomainEvent.html)
- [`desnormalizacao-mongo.md`](../02-persistencia/desnormalizacao-mongo.md) — a decisão de modelagem que criou a necessidade destes eventos
- [`ports-hexagonal.md`](./ports-hexagonal.md) — a porta de saída que a `ApplicationMessagePublisher` implementa
- [`concorrencia-e-atomicidade.md`](../02-persistencia/concorrencia-e-atomicidade.md) — a escrita que não passa pelo repositório, e por que ela exigiu uma terceira porta
- [`nosql-conceitos.md`](../02-persistencia/nosql-conceitos.md) — BASE, CAP e a consistência eventual em teoria
- [`arquitetura.md`](../00-visao-geral/arquitetura.md) — a mensageria entre serviços que ainda não existe
