# Concorrência e Atomicidade no MongoDB

> Como o estoque passou a mudar sem carregar o agregado: atualização condicional atômica, `$inc` como delta e o filtro que carrega a regra de negócio junto.
> Código real: `infrastructure/persistence/product/QuantityInStockAdjustmentMongoDBImpl.java`, `domain/product/StockService.java`, `domain/product/QuantityInStockAdjustment.java`.

> Este documento fecha uma pendência aberta desde a Fase 8: `quantityInStock` não tinha nenhuma operação pública de entrada ou saída, e produto criado pela API ficava travado em `0`.

---

## O problema

Dar baixa em estoque parece a coisa mais simples do mundo:

```java
Product product = productRepository.findById(productId).orElseThrow();

if (product.getQuantityInStock() >= quantity) {     // 1. confere
    product.withdraw(quantity);                      // 2. decide
    productRepository.save(product);                 // 3. grava
}
```

Está errado, e o erro não aparece em teste sequencial nenhum. Entre o passo 1 e o passo 3 cabe a requisição inteira de outra pessoa:

```
estoque = 1

Cliente A                          Cliente B
---------                          ---------
lê estoque = 1
                                   lê estoque = 1
1 >= 1 ✅ aprova
                                   1 >= 1 ✅ aprova
grava 0
                                   grava 0

estoque final = 0, mas saíram DUAS unidades de um estoque que tinha UMA
```

É o *lost update* clássico. A janela entre conferir e gravar é onde o dinheiro some — e ela existe **por o estado ter passado pela aplicação**.

---

## Três estratégias, e quando cada uma serve

| | Como detecta o conflito | Custo | Quando usar |
|---|---|---|---|
| **Lock otimista** (`@Version`) | a gravação falha se a versão mudou desde a leitura | carrega e regrava o agregado; alguém tem que tratar a falha e repetir | edição de campos que o usuário enxerga e revisa: nome, preço, descrição |
| **Atualização condicional** (`findAndModify`) | não detecta — **previne**: a regra vai no filtro e a operação é indivisível | uma ida ao banco, sem carregar nada | contadores, saldo, estoque, reserva de vaga |
| **Transação** | isolamento entre operações | exige replica set; a mais cara das três | escritas em coleções diferentes que precisam cair juntas |

O `product-catalog` usa as duas primeiras, em lugares diferentes e de propósito. O `@Version` continua valendo para `update`, `enable` e `disable`, que passam pelo repositório — ver [`product-catalog-mongo.md`](./product-catalog-mongo.md#6-lock-otimista--version).

**Por que estoque não usa lock otimista:** ele funcionaria, mas resolveria o problema errado. Lock otimista faz a segunda escrita *falhar* para alguém decidir o que fazer; num saque de estoque não há decisão a tomar — a operação é "subtraia 2", e ela só precisa ser aplicada uma vez, corretamente. Mandar o cliente repetir uma operação que sempre pode falhar de novo é empurrar complexidade para fora sem necessidade.

---

## A técnica: o filtro carrega a regra

```java
// infrastructure/persistence/product/QuantityInStockAdjustmentMongoDBImpl.java
Query query = Query.query(byId(productId)
        .and("quantityInStock").gte(quantity));

return changeStockQuantity(productId, quantity * -1, query);
```

A condição *"tem saldo"* **não é um `if` em Java** — ela é parte do filtro que o Mongo avalia. O banco casa o documento e aplica o update numa operação única e indivisível: não existe instante em que outra thread veja o documento "no meio" da operação.

É o mesmo raciocínio de um *compare-and-set*: em vez de perguntar e depois agir, você age condicionalmente e descobre pelo resultado se funcionou.

```java
Product before = mongoOperations.findAndModify(queryForUpdate, update,
        new FindAndModifyOptions().returnNew(false), Product.class);

if (before == null) {
    throw reasonForNoMatch(productId, delta);   // não existe OU sem saldo
}
```

`null` significa *"o filtro não casou"* — e nada foi alterado. Não há estado parcial para desfazer.

### `$inc` é um delta, não um valor

```java
Update update = new Update()
        .inc("quantityInStock", delta)
        .inc("version", 1)
        .set("updatedAt", OffsetDateTime.now());
```

Essa diferença é o segundo pilar da corretude. Compare:

| | Dois clientes concorrentes |
|---|---|
| `set("quantityInStock", lido - 2)` | cada um calcula sobre o que **leu**; o último a gravar apaga o outro |
| `inc("quantityInStock", -2)` | o servidor aplica cada incremento sobre o valor **corrente**; os dois contam |

Um `$inc` nunca perde escrita porque ele não carrega opinião sobre o valor anterior — só sobre a mudança.

> ⚠️ **`$inc` sozinho não impede estoque negativo.** Ele soma o delta ao que estiver lá, inclusive levando o saldo abaixo de zero. Quem impede o negativo é o `gte(quantity)` do **filtro**. Os dois trabalham juntos: o filtro decide *se*, o `$inc` decide *quanto*.

---

## ⚠️ "Atômico" não basta se a operação inteira não for

O `findAndModify` sempre foi atômico. Ainda assim, esta versão tinha uma corrida:

```java
// ANTES — duas idas ao banco, a primeira desprotegida
Document before = mongoOperations.aggregate(findOldProductQuantity, ...)
        .getUniqueMappedResult();                      // ← lê o "antes" aqui
Integer previousQuantity = before.getInteger("quantityInStock");

Product updated = mongoOperations.findAndModify(query, update,
        new FindAndModifyOptions().returnNew(true), Product.class);   // ← altera aqui
```

O `previousQuantity` vinha de uma leitura **separada e anterior**. Entre as duas, outra thread podia mexer no estoque — e o `previousQuantity` é exatamente o que decide se o evento sai:

```
estoque = 0

Thread A: increase(5)               Thread B: increase(5)
------------------------            ------------------------
lê previous = 0
                                    lê previous = 0
$inc → 5                            
  Result(prev=0, new=5)
  isRestocked() → true ✅
                                    $inc → 10
                                      Result(prev=0, new=10)
                                      isRestocked() → true ✅ ← ERRADO

dois ProductRestockedEvent para um reabastecimento,
e um previousQuantity (0) que nunca foi verdade para a thread B
```

A correção **encurta** o código. `returnNew(false)` devolve o documento como ele era antes do update, **na mesma operação que o alterou**:

```java
// DEPOIS — uma ida ao banco, atômica de ponta a ponta
Product before = mongoOperations.findAndModify(queryForUpdate, update,
        new FindAndModifyOptions().returnNew(false), Product.class);

int previousQuantity = before.getQuantityInStock();
return new Result(productId, previousQuantity, previousQuantity + delta);
```

O `newQuantity` é **calculado**, não relido: o `$inc` foi aplicado exatamente sobre o documento devolvido, então `previous + delta` é o que ficou gravado. Uma segunda leitura para "conferir" reabriria a janela que o `returnNew(false)` acabou de fechar.

> #### Nota de estudo: o instinto de ler antes
>
> Escrever `read → modify` é reflexo, e é por isso que essa corrida é tão comum. O ponto que vale guardar é que **`returnNew` não é um detalhe de conveniência sobre qual documento receber de volta** — ele é o que permite obter o estado anterior *sem uma leitura separada*. Sempre que um código precisa do "antes" e do "depois", a pergunta certa é se os dois vêm da mesma operação.

---

## Transição, não estado

O `Result` devolve os dois valores justamente porque o que interessa é a **mudança**:

```java
// domain/product/QuantityInStockAdjustment.java
public boolean isOutOfStock() {
    return newQuantity == 0 && previousQuantity != 0;
}

public boolean isRestocked() {
    return newQuantity > 0 && previousQuantity == 0;
}
```

Sem a segunda condição, `isOutOfStock()` seria verdade a **cada** consulta enquanto ninguém repõe — e viraria uma enxurrada de eventos idênticos. `previousQuantity != 0` é o que transforma *estado* em *acontecimento*.

| Antes | Depois | Evento |
|---|---|---|
| 50 | 40 | — |
| 10 | 0 | `ProductSoldOutEvent` |
| 0 | 0 | — (não houve transição) |
| 0 | 10 | `ProductRestockedEvent` |
| 40 | 50 | — |

---

## A versão que sobe sozinha

```java
.inc("version", 1)
```

Essa linha parece obrigatória e não é — o Spring Data faria o incremento de qualquer jeito. O bytecode de `spring-data-mongodb` conta a história: `MongoTemplate.doFindAndModify` chama `QueryOperations$UpdateContext.increaseVersionForUpdateIfNecessary`, que faz o equivalente a:

```java
if (entity.hasVersionProperty()) {
    String fieldName = entity.getRequiredVersionProperty().getFieldName();
    if (update != null && !update.modifies(fieldName)) {   // ← a condição que importa
        update.inc(fieldName);
    }
}
```

Ou seja: o framework incrementa **exceto se o update já mexer no campo**. A linha explícita é justamente o que o faz pular a dele — o efeito final é idêntico.

Ela fica por uma razão de legibilidade: a versão subindo é o que faz um `save()` concorrente vindo de outro caminho (`update`, `enable`) perceber que o documento mudou e falhar com `OptimisticLockingFailureException` em vez de sobrescrever o estoque em silêncio. Isso é importante demais para acontecer por efeito colateral do framework.

> **Verifique, não deduza.** Este comportamento não está óbvio na documentação; foi confirmado lendo o bytecode do jar em uso (`javap -c`). Quando uma garantia importa, a fonte é a implementação da versão que está no classpath.

---

## Onde o evento nasce, e por que aqui é diferente

A [Fase 12](../01-arquitetura-design/eventos-e-listeners.md) estabeleceu que `Product` estende `AbstractAggregateRoot` e que os eventos são publicados pelo Spring Data no `save()` do repositório.

**Esse mecanismo não serve aqui** — e a razão é a própria técnica deste documento: o ajuste de estoque não carrega nem salva o agregado. Não há `save()`, então não há publicação.

Daí a porta `DomainEventPublisher`, injetada no `StockService`:

```java
// domain/product/StockService.java
if (result.isOutOfStock()) {
    domainEventPublisher.publish(
            ProductSoldOutEvent.builder().productId(product.getId()).build());
}
```

O contraste vale mais que a regra:

| | Eventos da Fase 12 | Eventos de estoque |
|---|---|---|
| Onde são registrados | dentro do agregado (`registerEvent`) | no serviço de domínio |
| O que dispara a publicação | `productRepository.save()` | a chamada a `publish()` |
| Por quê | o agregado é carregado e salvo | o agregado nunca é carregado |

### `StockService` é um serviço de domínio

Estoque não virou método do `Product` porque **a operação não passa pelo agregado**. Um `product.withdraw(2)` exigiria ler, decidir em Java e salvar — exatamente o desenho que este documento existe para evitar. Quando o comportamento é de negócio mas não cabe em um agregado, ele vira um serviço de domínio; foi o que aconteceu.

O preço: `StockService` carrega um `@Service` do Spring dentro do pacote `domain`. Não é ideal, mas também não inaugura nada — `Product`, `Category` e os dois `Repository` já carregam anotações do Spring Data. A alternativa seria declarar o bean num `@Configuration` da `infrastructure`, deixando a classe limpa.

---

## Erro de negócio × falha técnica

Saldo insuficiente **não é** erro técnico: é uma regra sendo aplicada. E o cliente precisa distinguir isso de "o produto sumiu" e de "o banco caiu", porque a reação a cada um é diferente.

| Situação | Exceção | HTTP |
|---|---|---|
| Saldo insuficiente | `InsufficientStockException` | **422**, com pedido e disponível na mensagem |
| Produto inexistente | `ProductNotFoundException` | **404** |
| Mongo fora do ar, timeout | `DomainException` com a causa encadeada | 422 (do handler), com o stack trace preservado |

```java
// domain/product/StockService.java
try {
    return adjustment.get();
} catch (DomainException | DomainEntityNotFoundException e) {
    throw e;                                        // erro de negócio passa intacto
} catch (Exception e) {
    throw new DomainException(failureMessage, e);   // técnico: embrulha COM a causa
}
```

Um `catch (Exception e)` que devolve sempre a mesma mensagem e descarta o `e` custa duas coisas ao mesmo tempo: o cliente não sabe o que fazer, e quem for investigar não tem o stack trace do que realmente aconteceu.

O motivo da falha é decidido no adaptador, com uma checagem de existência que **só roda no caminho de erro** — o caminho feliz continua com uma única ida ao banco.

---

## Provando que funciona

Teste sequencial não prova nada aqui: um `findById → if → save` passaria em todos eles. A garantia só aparece com threads de verdade:

```java
// src/test/.../QuantityInStockAdjustmentConcurrencyIT.java
@Test
void shouldNotOversellUnderConcurrentWithdrawals() throws Exception {
    int threads = 20;
    int quantityEach = 5;          // estoque inicial = 50, cabem exatamente 10

    runConcurrently(threads, () -> { /* decrease, contando sucessos e falhas */ });

    assertThat(succeeded.get()).isEqualTo(10);
    assertThat(currentStock()).isZero();
    assertThat(currentStock()).isNotNegative();
}
```

O `CountDownLatch` é o que torna o teste útil: sem ele as threads sairiam escalonadas e a concorrência poderia nunca acontecer. Todas ficam bloqueadas no mesmo portão e são soltas juntas.

As três asserções cobrem coisas diferentes: **exatamente 10** sucessos (nem 9 nem 11), estoque final **zero** e **nunca negativo**. Uma implementação não atômica passa mais de 10 e derruba o saldo abaixo de zero.

Dá para ver o mesmo pela API:

```bash
ID=19274f99-e0d2-40b1-9b3a-912cb0982f11   # estoque 50
for i in $(seq 1 20); do
  curl -sX POST localhost:8083/api/v1/products/$ID/withdraw \
       -H 'Content-Type: application/json' -d '{"quantity":5}' -o /dev/null -w "%{http_code} " &
done; wait
# 10 respostas 204 e 10 respostas 422 — em qualquer ordem
```

> **Os testes rodam em outro Mongo.** Desde a Fase 14 os `*IT` sobem o próprio container (`TestContainerMongoDBConfig`), então nem chegam perto do banco de desenvolvimento. Antes disso a separação vinha do `src/test/resources/application-test-env.yml`, apontando para `product_catalog_test` — e foi por ficar de fora dessa migração que este teste de concorrência parou de rodar sem que ninguém notasse. Ver [`transacoes-mongo.md`](./transacoes-mongo.md) e [`carga-de-dados-mongo.md`](../04-infraestrutura/carga-de-dados-mongo.md).

---

## Armadilhas

1. **Ler, decidir e salvar é a armadilha inteira.** Se o estado passa pela aplicação entre a conferência e a gravação, existe uma janela — e ela só aparece sob carga.
2. **`$inc` não impede negativo.** Quem impede é o filtro. Os dois andam juntos.
3. **`set(valor)` perde escrita; `$inc(delta)` não.** A diferença é ter ou não opinião sobre o valor anterior.
4. **`returnNew(true)` obriga a ler o "antes" em outro lugar** — e esse outro lugar é uma corrida.
5. **`findAndModify` não passa pelo `AbstractAggregateRoot`.** Nenhum evento enfileirado no agregado é publicado por este caminho.
6. **A auditoria não acompanha.** `updatedAt` é setado à mão; `@LastModifiedDate` e `lastModifiedByUserId` não participam de escrita por `MongoOperations`.
7. **`null` do `findAndModify` tem dois significados.** "Não existe" e "não casou a condição" chegam iguais, e separá-los custa uma consulta a mais.
8. ~~**Não há transação.**~~ **Resolvido na Fase 14.** O ajuste passou a rodar dentro de um `@Transactional`, junto com o registro em `stock_movements` — e os listeners, por serem síncronos, entraram no mesmo commit. Isso exigiu trocar o Mongo de nó único por um replica set: ver [`transacoes-mongo.md`](./transacoes-mongo.md).

---

## Pendências registradas

- [x] ~~**Publicação de evento sem transação.**~~ Resolvido na Fase 14: os listeners de estoque são síncronos, portanto rodam **dentro** da transação — uma exceção no listener desfaz o ajuste. O que continua aberto é o degrau seguinte: um evento que precise sair para **fora** do serviço ainda não tem outbox. Ver [`transacoes-mongo.md`](./transacoes-mongo.md).
- [ ] **Transição é detectada por operação, não globalmente.** Duas instâncias da aplicação podem produzir transições concorrentes; o `Result` de cada uma é correto isoladamente, mas nada coordena os eventos entre processos.
- [ ] **A auditoria fica desatualizada no ajuste de estoque.** `lastModifiedByUserId` não é tocado; `updatedAt` é escrito à mão em vez de vir do `@LastModifiedDate`.
- [ ] **Sem contrato para `/restock` e `/withdraw`.** O OpenAPI declara os dois endpoints, mas não há contrato Spring Cloud Contract exercitando-os — diferente do resto da API.
- [ ] **O motivo da falha custa uma consulta extra.** Só no caminho de erro, mas é um round trip que uma implementação com `bulkWrite` ou `$expr` no update poderia evitar.
- [x] ~~**`quantityInStock` não tinha operação pública de entrada/saída.**~~ Resolvido nesta etapa: `POST /{productId}/restock` e `/withdraw`.
- [x] ~~**A leitura do "antes" ficava fora da operação atômica.**~~ Resolvido com `returnNew(false)`.

---

## Checklist de revisão

- [ ] Sei desenhar a linha do tempo do *lost update* com dois clientes
- [ ] Sei explicar por que lock otimista e atualização condicional resolvem problemas diferentes
- [ ] Entendo por que a condição de saldo vai no filtro e não num `if`
- [ ] Sei por que `$inc` não perde escrita e `set` perde
- [ ] Sei que `$inc` sozinho não impede estoque negativo
- [ ] Sei o que `returnNew(false)` resolve, e qual corrida existia sem ele
- [ ] Entendo por que `isOutOfStock` exige transição e não só estado
- [ ] Sei por que o incremento manual de `version` é redundante — e por que fica
- [ ] Sei por que os eventos de estoque não passam pelo `AbstractAggregateRoot`
- [ ] Sei por que `StockService` é um serviço de domínio e não um método do `Product`
- [ ] Entendo por que um teste sequencial não prova nada aqui

---

## Referências

- [MongoDB — `findAndModify`](https://www.mongodb.com/docs/manual/reference/command/findAndModify/)
- [MongoDB — `$inc`](https://www.mongodb.com/docs/manual/reference/operator/update/inc/)
- [MongoDB — Atomicity and Transactions](https://www.mongodb.com/docs/manual/core/write-operations-atomicity/)
- [Spring Data MongoDB — Optimistic Locking](https://docs.spring.io/spring-data/mongodb/reference/mongodb/template-crud-operations.html)
- [`product-catalog-mongo.md`](./product-catalog-mongo.md) — o lock otimista, que continua valendo nos outros caminhos
- [`eventos-e-listeners.md`](../01-arquitetura-design/eventos-e-listeners.md) — os três mecanismos de evento do serviço, e por que este precisou do terceiro
- [`desnormalizacao-mongo.md`](./desnormalizacao-mongo.md) — a outra escrita que não passa pelo repositório, e a tradução de `id` para `_id`
- [`carga-de-dados-mongo.md`](../04-infraestrutura/carga-de-dados-mongo.md) — o `auto-drop` e o banco separado de teste
