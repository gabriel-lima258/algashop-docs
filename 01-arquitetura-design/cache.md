# Cache: padrões, invalidação e os dois lados da chamada

> O mesmo dado guardado em quatro lugares — Mongo, Redis do catálogo, Redis do ordering e navegador — e o que cada cópia custa. Cache-aside, write-through, invalidação por TTL e por evicção, client-side × server-side.
> Código real: `application/product/query/ProductQueryService.java`, `application/product/management/ProductManagementApplicationService.java`, `infrastructure/cache/RedisCacheConfig.java` (product-catalog); `infrastructure/adapters/in/web/product/client/http/ProductCatalogApiClient.java` (ordering).

---

## O problema

Um catálogo é lido muitas vezes e escrito raramente. A mesma consulta de produto se repete o dia inteiro, e a resposta é quase sempre idêntica.

O custo disso não é só latência. Cada leitura ocupa uma conexão, consome CPU do banco e disputa I/O com as escritas — que são o que realmente precisa de garantia. Numa listagem popular, o banco passa a gastar a maior parte do fôlego respondendo perguntas cuja resposta ele já deu.

Cache é a decisão de **guardar a resposta em vez de recalculá-la**. E toda a dificuldade está na segunda metade da frase: uma resposta guardada é uma cópia, e cópia envelhece.

---

## Centralizado × distribuído — e o que foi escolhido

| | Como funciona | Custo |
|---|---|---|
| **Local** (Caffeine, `ConcurrentHashMap`) | cada instância tem o seu, na própria memória | duas instâncias divergem; invalidar numa não invalida na outra; reinício esvazia tudo |
| **Centralizado** (Redis, uma instância) | todas as instâncias falam com o mesmo | uma rede de distância; é um ponto único de falha |
| **Distribuído** (Redis Cluster, Hazelcast) | vários nós, dados particionados | complexidade de operação, rebalanceamento, topologia |

O projeto usa **centralizado**: um Redis, compartilhado pelos dois serviços que cacheiam.

A razão é a invalidação. Com cache local, `@CacheEvict` limpa a memória **da instância que atendeu a requisição** — as outras continuam servindo o valor antigo até o TTL. Isso transforma uma garantia ("depois de atualizar, ninguém mais vê o valor velho") numa probabilidade. Com cache centralizado, evictar é uma operação só, e vale para todo mundo.

O preço aceito é o ponto único de falha, e é justamente por isso que existe o [`ResilienceCacheErrorHandler`](#quando-o-cache-cai).

### Um Redis, dois serviços, dois bancos lógicos

```yaml
# product-catalog
spring.data.redis.database: 0

# algashop-ordering
spring.data.redis.database: 1
```

Bancos lógicos separam **namespace**: um `SCAN` no banco 0 não enxerga as chaves do banco 1, e um `FLUSHDB` num não afeta o outro. É o que impede colisão acidental de chave entre serviços.

> ⚠️ **O que eles não separam é recurso.** Memória, CPU e conexões são do processo Redis inteiro. Se o catálogo encher o `maxmemory`, o `allkeys-lru` despeja chaves *de qualquer banco* — inclusive as do ordering. Isolamento de namespace lido como isolamento de recursos é uma das confusões mais caras sobre Redis.

---

## Cache-aside × write-through

Os dois padrões convivem no mesmo serviço, e a diferença é **quem popula o cache**.

### Cache-aside (lazy loading) — `@Cacheable`

```java
// application/product/query/ProductQueryService.java
@Cacheable(cacheNames = CacheNames.PRODUCTS, key = "#productId")
ProductDetailOutput findById(UUID productId);
```

Olha o cache; se achar, devolve e **o método nem roda**; se não achar, roda, guarda o retorno e devolve. O cache é preenchido pela leitura — a primeira paga, as seguintes não.

É o padrão certo quando não se sabe o que será lido. O cache guarda só o que alguém realmente pediu.

### Write-through — `@CachePut`

```java
// application/product/management/ProductManagementApplicationService.java
@CachePut(cacheNames = CacheNames.PRODUCTS, key = "#result.id",
          condition = "#input.enabled == true")
public ProductDetailOutput create(ProductInput input) {
    Product product = mapToProduct(input);
    productRepository.save(product);
    return mapper.convert(product, ProductDetailOutput.class);
}
```

Grava no banco **e** no cache, na mesma operação. O produto já nasce quente: a primeira leitura não paga miss.

Diferença essencial em relação ao `@Cacheable`: `@CachePut` **sempre executa o método** e guarda o retorno. Ele não pergunta ao cache — ele o atualiza.

> #### Nota de estudo: uma decisão de cache mudou a assinatura de um método
>
> Repare no `key = "#result.id"`. Para o SpEL alcançar `#result.id`, o método precisa **retornar** algo com `id`. Antes, `create` devolvia `UUID` e `update` devolvia `void`; os dois passaram a devolver `ProductDetailOutput`.
>
> Isso obrigou o `ProductController` a parar de reconsultar o query service depois de escrever, e obrigou o `ProductBase` dos contract tests a trocar `doNothing()` por `thenReturn(...)`.
>
> É a natureza do `@CachePut`: ele cacheia **o retorno**, então o retorno tem que ser aquilo que se quer cachear. Vale saber antes de anotar — a alternativa seria cachear pelo argumento e reler, que é justamente o que se estava tentando evitar.

---

## Client-side × server-side

O mesmo produto acaba guardado em quatro lugares, e cada camada tem seu próprio relógio:

```
navegador          Cache-Control: max-age=60   ← client-side (HTTP)
   ↓
ordering/Redis     TTL 5 min                   ← client-side (o ordering cacheia
   ↓                                              a resposta do catálogo)
catalog/Redis      TTL 1 min                   ← server-side
   ↓
MongoDB                                        ← a verdade
```

### Server-side — o serviço cacheia a própria resposta

É o `@Cacheable`/`@CachePut` do `product-catalog`, descrito acima. Quem escreve o dado é o mesmo serviço que o cacheia, então ele **sabe** quando a entrada ficou velha — e por isso pode usar `@CacheEvict`.

### Client-side por cabeçalho HTTP

```java
// presentation/ProductController.java
return ResponseEntity.ok()
        .cacheControl(CacheControl.maxAge(Duration.ofMinutes(1)).cachePublic())
        .eTag("product:id" + product.getId() + ":v:" + product.getVersion())
        .lastModified(product.getUpdatedAt().toInstant())
        .body(product);
```

Aqui não se guarda nada — apenas se **autoriza** o cliente a guardar, e se dá a ele como perguntar "mudou?" sem baixar o corpo de novo.

O `ETag` sai do `@Version` do documento. É a escolha certa de validador: a versão só muda quando o Mongo grava, então ela é exatamente "este produto mudou". Um hash do corpo daria o mesmo resultado por muito mais trabalho.

A listagem de categorias vai um passo além e responde **304** quando nada mudou:

```java
// presentation/CategoryController.java
OffsetDateTime lastModified = categoryQueryService.lastModified();
if (webRequest.checkNotModified(lastModified.toInstant().toEpochMilli())) {
    return ResponseEntity.status(HttpStatus.NOT_MODIFIED).build();
}
```

`checkNotModified` compara com o `If-Modified-Since` que o cliente enviou. Batendo, a resposta é `304` **sem corpo** — a consulta ao banco nem chega a acontecer, e a rede transporta cabeçalhos em vez de uma página inteira.

O `lastModified()` é uma agregação `max(updatedAt)` sobre a coleção. E ele **não é cacheado**, de propósito: cachear o carimbo que decide se o cliente pode reusar seria cachear o próprio critério de invalidação.

### Client-side entre serviços

```java
// ordering — infrastructure/.../ProductCatalogApiClient.java
@Cacheable(cacheNames = "algashop:product-catalog-api:v1", key = "#productId")
@GetExchange(value = "/api/v1/products/{productId}", accept = "application/json")
ProductResponse getById(@PathVariable UUID productId);
```

O `ordering` cacheia a **resposta HTTP** do catálogo. Funciona porque o bean é um proxy JDK criado por `HttpServiceProxyFactory`, e o auto-proxy do `@EnableCaching` o envelopa lendo a anotação da interface.

> #### Nota de estudo: onde o cache entra na pilha de resiliência
>
> Na Fase 16 esse mesmo método ganhou bulkhead e circuit breaker em volta, e a ordem real dos proxies **não** é a ordem das anotações:
>
> ```
> @ConcurrencyLimit → @Cacheable → getById → circuitBreaker.run → retry → HTTP
> ```
>
> O bulkhead fica **por fora** do cache (ele usa `setBeforeExistingAdvisors(true)`), e o circuito **por dentro**. Duas consequências que só aparecem lendo nessa ordem:
>
> - **Cache hit não toca o circuito.** Uma resposta servida do Redis não conta como sucesso nem como falha — o circuito só enxerga o que sai para a rede. Um circuito fechado não prova que a dependência está viva; prova que a última chamada real deu certo.
> - **Cache hit ainda ocupa uma vaga do bulkhead.** As 10 permissões são consumidas por qualquer chamada ao método, inclusive as que nem saem da JVM. Sob carga com taxa de acerto alta, o limite pode ser atingido sem uma única ida ao catálogo.
>
> Ver [`resiliencia.md`](./resiliencia.md).

> #### Nota de estudo: quem cacheia dado dos outros só tem TTL
>
> Esta é a diferença que define o lado cliente. No catálogo, quem escreve o produto é quem o cacheia — então dá para evictar no momento exato da mudança.
>
> No `ordering`, o dado é de outro serviço, e **ele não fica sabendo quando muda**. Não há `@CacheEvict` possível: não existe evento, não existe callback, não existe nada. Sobra o TTL — e o TTL não é uma escolha de performance, é a única ferramenta disponível.
>
> É por isso que a pergunta "qual TTL manda?" tem uma resposta desconfortável: **manda o maior**. Adiantar de nada reduzir o TTL do Mongo para o catálogo se o ordering guarda a resposta por cinco minutos e o navegador por mais um. A idade máxima de um dado é a soma das camadas, não a menor delas.

---

## Invalidação

Três mecanismos em uso, e um que **não** está.

### TTL — o piso

Toda entrada expira. É o mecanismo que não depende de ninguém lembrar de nada, e por isso é o único que nunca falha silenciosamente. Tudo o mais é otimização em cima dele.

### `@CacheEvict` — a invalidação explícita

```java
@Caching(evict = {
        @CacheEvict(cacheNames = CacheNames.CATEGORIES_FILTER, key = "'default'"),
        @CacheEvict(cacheNames = CacheNames.CATEGORIES, key = "#categoryId")
})
public void disable(UUID categoryId) { ... }
```

A regra prática: **todo método que muda estado cacheado precisa dizer o que fazer com o cache — inclusive para dizer "nada".** Três evicções faltavam nesta leva, e cada uma tinha um sintoma próprio:

| Faltava em | O que o cache passava a mentir |
|---|---|
| `Category.disable` | categoria desabilitada continuava aparecendo na listagem, com `enabled: true` |
| `Product.enable` | produto reabilitado continuava saindo como `enabled: false` |
| `restock` / `withdraw` | `inStock` desatualizado — o campo que alguém consulta antes de comprar |

O caso do estoque é o mais instrutivo, porque era o mais fácil de não ver: `restock` e `withdraw` **não carregam nem salvam o produto** — o ajuste vai direto ao banco por `findAndModify`. Nada nesses métodos *parece* mexer no produto.

> ⚠️ **`@CacheEvict` e `@Transactional` no mesmo método: a ordem não é garantida.** Os dois interceptadores registram seus advisors com `LOWEST_PRECEDENCE`, e o desempate fica por conta da ordem de registro.
>
> O que **é** garantido: `beforeInvocation = false` (o padrão) só evicta se o método retornar sem exceção. Saque recusado por saldo insuficiente não evicta nada — correto, já que o banco também não mudou.
>
> O que machuca é evicção **antes** do commit: abre uma janela em que outra thread lê o banco ainda no valor antigo e **repopula** o cache com ele — dado velho de volta, agora com o TTL inteiro pela frente.

### `@CachePut` — invalidação por sobrescrita

Atualizar a entrada em vez de removê-la. Evita o miss seguinte, ao custo de gravar algo que talvez ninguém leia.

### O que não existe: invalidação por evento

O `product-catalog` já tem um mecanismo de eventos, e ele **não** conversa com o cache:

```
PUT /categories/{id}
   → grava a categoria
   → @CacheEvict limpa algashop:categories:v1 e o filtro    ✅
   → publica CategoryUpdatedEvent
        → CategoryEventListener (@Async)
             → updateMulti reescreve a cópia da categoria
               dentro de TODOS os produtos daquela categoria
             → e NADA invalida algashop:products:v1          ❌
```

Um produto em cache continua servindo o **nome antigo da categoria** até o TTL expirar. Não é hipótese: é o desenho atual.

**A resposta escolhida foi encurtar o TTL, não acoplar o listener ao cache.** O TTL de `algashop:products:v1` passou de 5 para 1 minuto — que é também o `max-age` que o `ProductController` publica, então as duas camadas passaram a contar a mesma história.

A troca, dita por inteiro: o listener publicaria uma evicção precisa e a janela iria a zero, ao custo de a infraestrutura de eventos passar a conhecer a de cache, e de um produto ficar sem cache logo depois de qualquer renomeação de categoria. Um minuto de nome de categoria desatualizado num catálogo é um preço baixo. **Numa aplicação onde não fosse, a conta daria outro resultado — e o ponto é que essa é uma decisão de produto, não de engenharia.**

---

## Só o filtro default é cacheável

```java
// application/category/query/CategoryFilter.java
public boolean isCacheable() {
    return this.equals(defaultFilter());   // name=null, enabled=true, page=0,
}                                          // size=15, sort NAME ASC
```

Usado em dois lugares: no `condition` do `@Cacheable` e no controller, para decidir se emite cabeçalhos HTTP de cache.

Cachear listagem com filtro livre é **armadilha de cardinalidade**. Cada combinação de nome, `enabled`, página, tamanho e ordenação vira uma chave própria; com poucos parâmetros já são milhares de combinações, quase todas pedidas uma vez só. O Redis enche de entradas que nunca são lidas de novo, a memória acaba, o `allkeys-lru` começa a despejar — e despeja também o que era útil.

O filtro default é o oposto: é o que a home pede, o que quase toda visita faz sem tocar em nada. Uma chave só, com reuso altíssimo.

**Cache paga quando há reuso.** A listagem de produtos, que tem busca textual e faixa de preço, não é cacheada por essa mesma razão.

---

## Serialização: JDK, não JSON

O `RedisCacheConfig` parte de `RedisCacheConfiguration.defaultCacheConfig()` e **não troca o serializador de valores** — vale o padrão, que é a serialização nativa do Java.

Daí vem o `implements Serializable` espalhado pelos DTOs. E a exigência é **transitiva**: todo campo também precisa ser serializável, o que arrastou o `CategoryMinimalOutput` e o `PageModel` junto.

| | JDK | JSON (`GenericJackson2JsonRedisSerializer`) |
|---|---|---|
| Inspecionar com `redis-cli GET` | binário opaco | legível |
| Interoperar com outra linguagem | impossível | natural |
| Acrescentar um campo ao DTO | quebra as entradas antigas | tolerante |
| Configuração | zero | precisa lidar com tipos polimórficos e `@class` |

> ⚠️ **Nenhum DTO declara `serialVersionUID`.** Sem ele, a JVM calcula um a partir da assinatura da classe — então **acrescentar um campo muda o identificador**, e as entradas gravadas antes passam a estourar `InvalidClassException` na leitura.
>
> O `ResilienceCacheErrorHandler` transforma isso em log em vez de erro, e o TTL curto limpa o resto — mas é sorte, não desenho. O sufixo `:v1` nos nomes de cache existe justamente para isso: numa mudança de DTO, subir para `:v2` troca o namespace inteiro de uma vez.

---

## Quando o cache cai

```java
// infrastructure/cache/ResilienceCacheErrorHandler.java
@Override
public void handleCacheGetError(RuntimeException exception, Cache cache, Object key) {
    logWarn(exception, cache, key, "GET");   // engole e segue para o banco
}
```

Sem handler, o padrão do Spring é **propagar**: Redis fora do ar derruba a requisição. Isso inverte a razão de existir do cache — ele foi posto ali para o sistema aguentar mais carga, e passaria a ser um ponto de falha novo.

Todos os quatro métodos engolem e seguem. É **fail-open**: degrada em performance, não em disponibilidade.

A distinção entre `WARN` e `ERROR` é o detalhe que vale copiar:

- Quase tudo ali é problema de **infraestrutura** — Redis reiniciando, rede oscilando — e some sozinho. `WARN`, sem stacktrace, senão o log vira ruído durante a indisponibilidade.
- `SerializationException` no PUT é problema de **código**: alguém tentou cachear algo que não implementa `Serializable`. Não melhora sozinho, vai repetir em toda escrita. `ERROR` **com** stacktrace, que é a única forma de descobrir qual campo da árvore de objetos não é serializável.

No `ordering` a consequência de não ter handler seria pior, e por isso ele ganhou um igual: o que está cacheado ali é uma **chamada HTTP**. Sem handler, uma queda do Redis derrubaria a criação de pedido com o catálogo de pé, respondendo normalmente.

---

## Armadilhas

1. **Cache local invalida só a própria instância.** Com duas réplicas, `@CacheEvict` vira probabilidade.
2. **Banco lógico separa namespace, não recursos.** O `maxmemory` é do processo inteiro.
3. **`@CachePut` cacheia o retorno**, então o retorno tem que ser o que se quer cachear — e isso muda assinaturas.
4. **Todo método que muda estado cacheado precisa decidir sobre o cache.** Os que não parecem mexer no agregado são os mais fáceis de esquecer.
5. **A ordem entre `@CacheEvict` e `@Transactional` não é garantida.**
6. **A idade máxima de um dado é a soma das camadas**, não a menor.
7. **Cachear listagem com filtro livre enche o Redis de chaves lidas uma vez só.**
8. **Serialização JDK sem `serialVersionUID`** transforma qualquer campo novo em `InvalidClassException`.
9. **`@EnableCaching` ausente não dá erro.** Toda anotação de cache vira enfeite, silenciosamente.

---

## ⚠️ O estado real: o cache não popula na aplicação rodando

Tudo o que está descrito acima é o **desenho**. A verificação de ponta a ponta mostrou que ele não está funcionando, e o que se sabe até aqui é isto:

Com o `product-catalog` de pé no perfil `development`, depois de um `GET /api/v1/products/{id}`:

```
DBSIZE                → 0
INFO keyspace         → vazio
INFO clients          → connected_clients: 1   (o próprio redis-cli)
```

A aplicação **não chega a abrir conexão** com o Redis. O que já foi descartado:

| Hipótese | Como foi descartada |
|---|---|
| Redis inacessível ou senha errada | `redis-cli -a algashop ping` responde, e sem senha é recusado |
| `RedisCacheConfig` não registrado | O relatório de condições mostra `RedisCacheConfig matched` |
| `@EnableCaching` sem efeito | O bean `cacheInterceptor` existe (`CacheAutoConfiguration` o encontra) |
| Tipo de cache errado | `RedisCacheConfiguration matched — REDIS cache type` |
| Falha de conexão engolida pelo handler | Zero ocorrências de `Cache GET error` no log |
| `@Cacheable` na interface não ser herdada pelo proxy CGLIB | Anotar a **implementação** também não populou o cache |

O último item é o que derruba a explicação mais provável. A infraestrutura de cache está montada, o interceptador existe, e ainda assim nenhum método anotado é interceptado.

### A Fase 17 acrescentou uma evidência — e ela reformula o problema

O health check trouxe um `CustomRedisCacheHealthIndicator` que faz um `ping()` explícito no Redis. Ele deveria ser o teste mais simples possível de conectividade. Medido com o `ordering` no ar:

```
1. Redis PARADO, três leituras seguidas do /actuator/health:
   cache = UP, UP, UP

2. CONFIG RESETSTAT, dois GET /actuator/health, e o servidor contando comandos:
   PINGs vindos da aplicação: 0

3. CLIENT LIST no Redis, com o serviço rodando:
   1 cliente — o próprio redis-cli. A aplicação: nenhuma conexão.
```

Ou seja: **um `ping()` explícito também não abre conexão, e também não falha.** Isso muda a hipótese. O problema não parece estar no proxy de cache nem nas anotações — está mais embaixo, no cliente Redis desta aplicação, que retorna sucesso sem falar com o servidor.

Continua sem causa raiz identificada, mas agora com duas evidências apontando para o mesmo lugar, e um caminho de investigação bem mais estreito que o da Fase 15. Ver [`health-checks.md`](../04-infraestrutura/health-checks.md).

> **Isto fica em aberto, e é a razão de a pendência seguinte ser a mais importante da lista.** Um teste automatizado teria pegado isso no minuto zero — e a ausência dele é o que permitiu a leva inteira ser escrita, revisada e documentada com o cache desligado o tempo todo.
>
> Vale também como caso exemplar do que este documento já dizia em outro lugar: **cache mal configurado não quebra nada**. A API responde certo, os 56 testes passam, e o único sintoma é uma performance que nunca melhorou.

---

## Pendências registradas

- [ ] **O cache não popula na aplicação rodando** — ver a seção acima. Causa raiz não identificada; as hipóteses óbvias foram descartadas com evidência.
- [ ] **Nenhum teste automatizado cobre o cache.** Nenhum `*IT` exercita `@Cacheable`, `@CacheEvict` ou `@CachePut`, e `spring.cache.type` nem sequer é `redis` no perfil de teste — o `@ConditionalOnProperty` não bate e `@EnableCaching` não é registrado. É a pendência mais séria desta etapa: a corretude do cache hoje depende de leitura de código.
- [ ] **Invalidação por evento não existe.** A propagação assíncrona da categoria não alcança `algashop:products:v1`; a janela é de 1 minuto.
- [ ] **Ordem entre cache e transação indefinida.** Fechar de vez pediria `@TransactionalEventListener(AFTER_COMMIT)` publicando a evicção, ou `@EnableCaching(order = ...)`.
- [ ] **Cache só existe no perfil `development`.** Em `docker` e `production` não há `spring.cache.type`, então a aplicação roda sem cache nenhum — e sem nada avisando.
- [ ] **`spring.data.redis.timeout: 600` é 600 milissegundos**, não segundos. É um `Duration` sem unidade. Provavelmente não foi o pretendido.
- [ ] **Sem `serialVersionUID` em nenhum DTO cacheado**, e `PageModel<T>` não exige `T extends Serializable` — a garantia é de convenção.
- [ ] **TTLs de Redis e `max-age` divergem nas categorias** — 1 minuto no Redis, 5 no HTTP. Nos produtos foram alinhados nesta etapa; nas categorias não.

---

## Checklist de revisão

- [ ] Sei dizer a diferença entre cache-aside e write-through, e quando cada um serve
- [ ] Entendo por que `@CachePut` obrigou `create` a mudar de assinatura
- [ ] Sei por que quem cacheia dado de outro serviço só tem TTL como invalidação
- [ ] Sei explicar por que a idade máxima de um dado é a soma das camadas
- [ ] Entendo por que só o filtro default é cacheável
- [ ] Sei o que o `ETag` derivado do `@Version` garante, e por que `lastModified()` não é cacheado
- [ ] Sei por que `implements Serializable` foi necessário, e o que se perderia com JSON
- [ ] Entendo por que erro de cache é engolido e por que `SerializationException` é a exceção
- [ ] Sei por que banco lógico não isola memória
- [ ] Sei o que acontece se `@EnableCaching` sumir — e por que nada avisa

---

## Referências

- [Spring Framework — Cache Abstraction](https://docs.spring.io/spring-framework/reference/integration/cache.html)
- [Spring Data Redis — Redis Cache](https://docs.spring.io/spring-data/redis/reference/redis/redis-cache.html)
- [MDN — HTTP Caching](https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching)
- [RFC 9111 — HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111)
- [`redis.md`](../04-infraestrutura/redis.md) — o Redis na prática: compose, eviction, inspeção
- [`eventos-e-listeners.md`](./eventos-e-listeners.md) — a propagação assíncrona que não alcança o cache
- [`product-catalog-mongo.md`](../02-persistencia/product-catalog-mongo.md) — o `@Version` que vira `ETag`
- [`transacoes-mongo.md`](../02-persistencia/transacoes-mongo.md) — a transação com que a evicção convive
