# Consultas dinâmicas no MongoDB com `Criteria`

> Como o `product-catalog` monta um filtro de listagem com 9 parâmetros opcionais sem escrever 512 consultas.
> Código real: `infrastructure/persistence/product/ProductQueryServiceImpl.java`.

---

## O problema

A tela de catálogo precisa filtrar produto por: termo de busca, faixa de preço, faixa de data, categoria, ativo, em estoque e com desconto. Todos opcionais, em qualquer combinação.

A abordagem ingênua é uma consulta por combinação:

```java
// não faça isso
List<Product> findByEnabled(Boolean enabled);
List<Product> findByEnabledAndBrand(Boolean enabled, String brand);
List<Product> findByEnabledAndBrandAndSalePriceBetween(...);
// ... e assim por diante
```

Com 9 filtros booleanos de presença, são 2⁹ = **512 combinações**. Não dá.

A saída é montar a consulta **em tempo de execução**, acrescentando uma condição por filtro preenchido. No MongoDB isso é a dupla `Query` + `Criteria`.

---

## `Query` e `Criteria` — as duas peças

| Peça | Papel | Analogia SQL |
|---|---|---|
| `Criteria` | uma condição isolada | um pedaço do `WHERE` |
| `Query` | o conjunto de condições + ordenação + paginação | a consulta inteira |

```java
Query query = new Query();
query.addCriteria(Criteria.where("enabled").is(true));
query.addCriteria(Criteria.where("quantityInStock").gt(0));
```

Isso gera:

```javascript
{ "enabled": true, "quantityInStock": { "$gt": 0 } }
```

**Detalhe importante:** dois `addCriteria` viram um `AND` implícito — no BSON, chaves diferentes no mesmo objeto já significam "todas devem casar". Não existe `$and` explícito no documento gerado.

### Comparando com a JPA Criteria API

O `ordering` faz a mesma coisa com JPA — ver [`paginacao.md`](./paginacao.md). Vale o paralelo:

| | JPA Criteria API | Spring Data MongoDB |
|---|---|---|
| Objeto raiz | `CriteriaQuery<T>` | `Query` |
| Condição | `Predicate` | `Criteria` |
| Fábrica | `CriteriaBuilder` | métodos estáticos de `Criteria` |
| Combina | `builder.and(p1, p2)` | `addCriteria` sucessivos |
| Executa | `entityManager.createQuery(...)` | `mongoOperations.find(query, Product.class)` |
| Type-safe? | sim (metamodel) | **não** — o campo é `String` |

Essa última linha é a armadilha do lado Mongo: `Criteria.where("salePrise")` compila normalmente e simplesmente não acha nada. Renomear um campo do `Product` não quebra o build — quebra em produção, silenciosamente.

---

## A hierarquia de filtros

Antes da consulta, os parâmetros. Três níveis, cada um com uma responsabilidade:

```
PageFilter                    page, size          (todo mundo pagina)
    ↑
SortablePageFilter<T>         sortByProperty, sortDirection
    ↑
ProductFilter                 term, priceFrom, categoriesId, ...
```

```java
// application/util/PageFilter.java
public class PageFilter {
    private int size = 15;   // default evita devolver a coleção inteira
    private int page = 0;
}
```

O nível do meio é o interessante:

```java
// application/util/SortablePageFilter.java
public abstract class SortablePageFilter<T> extends PageFilter {
    private T sortByProperty;
    private Sort.Direction sortDirection;

    public abstract T getSortByPropertyOrDefault();
    public abstract Sort.Direction getSortDirectionOrDefault();
}
```

O genérico `<T>` é preenchido com um **enum** definido pelo filtro concreto:

```java
// application/product/query/ProductFilter.java
public class ProductFilter extends SortablePageFilter<ProductFilter.SortType> {

    public enum SortType {
        ADDED_AT("addedAt"),
        SALE_PRICE("salePrice");

        private final String propertyName;
    }
}
```

Por que isso importa: se `sortByProperty` fosse `String`, o cliente poderia mandar `?sortByProperty=senhaDoAdmin` e o Mongo tentaria ordenar por um campo qualquer. Com enum, **o conjunto de campos ordenáveis é fechado** — valor fora da lista nem chega a ser bindado. É a mesma ideia de uma allowlist.

Os dois métodos abstratos existem para forçar cada filtro concreto a declarar seu default, evitando `NullPointerException` quando o cliente não manda nada de ordenação.

---

## Binding: o filtro inteiro como parâmetro

Antes eram dois `@RequestParam` soltos:

```java
// como era
@GetMapping
public PageModel<ProductSummaryOutput> filter(
        @RequestParam(required = false) Integer size,
        @RequestParam(required = false) Integer number) {
    return productQueryService.filter(size, number);
}
```

Agora é um **command object**:

```java
// presentation/ProductController.java — como ficou
@GetMapping
public PageModel<ProductSummaryOutput> filter(ProductFilter filter) {
    return productQueryService.filter(filter);
}
```

Sem anotação nenhuma. Quando o parâmetro não é um tipo simples nem tem `@RequestBody`, o Spring MVC trata como objeto de comando e preenche os campos pelo nome, direto da query string:

```
GET /api/v1/products?enabled=true&priceFrom=100&priceTo=500&size=20&page=0
```

Ganho concreto: acrescentar um filtro novo é acrescentar um campo no `ProductFilter`. A assinatura do controller e do service não mudam.

---

## Montagem incremental

O coração do filtro dinâmico cabe em quatro linhas de padrão:

```java
private Query queryWith(ProductFilter filter) {
    Query query = new Query();

    if (filter.getEnabled() != null) {
        query.addCriteria(Criteria.where("enabled").is(filter.getEnabled()));
    }
    // ... um bloco desses por filtro

    return query;
}
```

`null` significa **"não filtra por isso"** — não significa "filtra por null". É a diferença entre "tanto faz se está ativo" e "traga os que têm `enabled: null`".

Nos filtros booleanos isso dá três estados úteis com um só parâmetro:

| `?enabled=` | Resultado |
|---|---|
| ausente | ativos **e** inativos |
| `true` | só os ativos |
| `false` | só os inativos |

---

## Cada filtro e o documento que ele gera

| Campo do filtro | Criteria em Java | Query BSON gerada |
|---|---|---|
| `enabled` | `where("enabled").is(v)` | `{ enabled: true }` |
| `addedAtFrom` + `addedAtTo` | `where("addedAt").gte(a).lte(b)` | `{ addedAt: { $gte: a, $lte: b } }` |
| `priceFrom` + `priceTo` | `where("salePrice").gte(a).lte(b)` | `{ salePrice: { $gte: a, $lte: b } }` |
| `inStock` | `where("quantityInStock").gt(0)` | `{ quantityInStock: { $gt: 0 } }` |
| `categoriesId` | `where("categoryId").in(ids)` | `{ categoryId: { $in: [...] } }` |
| `hasDiscount` | `AggregationExpressionCriteria.whereExpr(...)` | `{ $expr: { $lt: ["$salePrice", "$regularPrice"] } }` |
| `term` | `new Criteria().orOperator(...)` | `{ $or: [ {name: /x/i}, {brand: /x/i}, {description: /x/i} ] }` |

### Intervalos: por que o `if/else` em vez de dois `if`

```java
if (filter.getPriceFrom() != null && filter.getPriceTo() != null) {
    query.addCriteria(Criteria.where("salePrice")
            .gte(filter.getPriceFrom())
            .lte(filter.getPriceTo()));
} else {
    if (filter.getPriceFrom() != null) {
        query.addCriteria(Criteria.where("salePrice").gte(filter.getPriceFrom()));
    } else if (filter.getPriceTo() != null) {
        query.addCriteria(Criteria.where("salePrice").lte(filter.getPriceTo()));
    }
}
```

Parece verboso, e é — mas tem motivo. Se fossem dois `addCriteria` independentes no **mesmo campo**:

```java
// quebra em tempo de execução
query.addCriteria(Criteria.where("salePrice").gte(from));
query.addCriteria(Criteria.where("salePrice").lte(to));
// InvalidMongoDbApiUsageException: Due to limitations of the com.mongodb.BasicDocument,
// you can't add a second 'salePrice' criteria.
```

O documento BSON é um mapa: não existe a mesma chave duas vezes. Os dois operadores precisam ser **encadeados no mesmo `where`** para caírem dentro do mesmo objeto. Daí o `if/else`.

### `categoryId`: filtrar sem carregar a categoria

```java
query.addCriteria(Criteria.where("categoryId").in((Object[]) filter.getCategoriesId()));
```

O campo procurado é o `categoryId` que o `@DocumentReference` grava dentro do documento de produto — não a `Category` inteira. Filtrar por categoria não custa nenhuma leitura extra: o dado já está no documento. Ver [`product-catalog-mongo.md`](./product-catalog-mongo.md#2-relacionamento-por-referência--documentreference).

---

## Comparar dois campos do mesmo documento

`hasDiscount` significa "o preço de venda está abaixo do preço cheio" — `salePrice < regularPrice`. Não existe valor literal na comparação: os **dois lados são campos**.

Isso é um limite real do `Criteria` comum. `where("salePrice").lt(x)` só aceita um valor em `x`; passar o nome de outro campo não funciona, porque o Mongo compararia com a string `"regularPrice"`.

A saída é o operador `$expr`:

```java
query.addCriteria(AggregationExpressionCriteria.whereExpr(
        ComparisonOperators.valueOf("$salePrice").lessThan("$regularPrice")
));
```

```javascript
{ "$expr": { "$lt": ["$salePrice", "$regularPrice"] } }
```

O `$` antes do nome não é sintaxe do Java — é como o Mongo diz "o valor deste campo", em vez do texto literal.

> ⚠️ **Custo:** `$expr` roda para cada documento e **não usa índice**. Numa coleção grande, esse filtro sozinho força varredura completa. A alternativa é o `Product` já gravar um campo `hasDiscount` calculado na escrita — mais espaço em disco, consulta indexável. O agregado, aliás, já mantém `discountPercentageRounded` exatamente com essa lógica.

---

## Busca textual por regex

```java
private static final String flexibleRegex = "(?i)%s";

if (StringUtils.isNotBlank(filter.getTerm())) {
    String regexExpression = String.format(flexibleRegex, filter.getTerm());
    query.addCriteria(
            new Criteria().orOperator(
                    Criteria.where("name").regex(regexExpression),
                    Criteria.where("brand").regex(regexExpression),
                    Criteria.where("description").regex(regexExpression)
            )
    );
}
```

Três pontos:

1. **`orOperator` agrupa** — os três viram um único criteria com `$or`. Se fossem três `addCriteria`, virariam `AND`, e o produto precisaria ter o termo nos três campos ao mesmo tempo.
2. **`(?i)`** liga o modo case-insensitive dentro da própria regex, sem precisar do segundo argumento de `regex()`.
3. **O `%s` é do `String.format`**, não do Mongo — o projeto tem duas regex prontas:

| Constante | Padrão | `?term=note` acha "Notebook"? |
|---|---|---|
| `flexibleRegex` | `(?i)%s` | **sim** — casa em qualquer posição |
| `regularRegex` | `(?i)(?<= \|^)%s(?= \|$)` | não — exige palavra inteira |

### Regex vs. índice de texto

Regex sem âncora no início (`/note/` em vez de `/^note/`) **não usa índice** — o Mongo varre a coleção. A alternativa nativa é o índice de texto:

```javascript
db.products.createIndex({ name: "text", brand: "text", description: "text" })
// consulta: { $text: { $search: "notebook gamer" } }
```

Índice de texto é muito mais rápido e faz stemming, mas casa **palavras inteiras** — busca incremental ("note" enquanto o usuário digita) não funciona. Para o volume deste projeto, regex está de bom tamanho; a decisão mudaria com escala.

---

## Paginação: contar antes de paginar

```java
// 1. monta os criterias de filtro
Query query = queryWith(filter);

// 2. conta o total ANTES de paginar — mesma query, sem skip/limit
long totalItems = mongoOperations.count(query, Product.class);

// 3. aplica ordenação + skip/limit sobre a mesma query filtrada
Sort sort = sortWith(filter);
PageRequest pageRequest = PageRequest.of(filter.getPage(), filter.getSize(), sort);
Query pagedQuery = query.with(pageRequest);

if (totalItems > 0) {
    products = mongoOperations.find(pagedQuery, Product.class);
    totalPages = (int) Math.ceil((double) totalItems / pageRequest.getPageSize());
}
```

**A ordem importa.** O `count` acontece com a query ainda sem `skip`/`limit` — ele precisa saber quantos documentos casam com o filtro **inteiro**, não quantos cabem na página. Contar depois do `.with(pageRequest)` devolveria no máximo `size`, e o `totalPages` sairia sempre 1.

Duas idas ao banco por página é o preço normal de uma listagem paginada — o `ordering` faz igual com JPA ([`paginacao.md`](./paginacao.md)).

**O `Math.ceil` e o cast para `double`:** sem o cast, `totalItems / pageSize` é divisão inteira e 25 itens em páginas de 10 dariam 2 páginas em vez de 3. Os 5 do resto ficariam inalcançáveis.

**O `if (totalItems > 0)`:** se nada casou, nem vale ir ao banco de novo — devolve lista vazia direto.

---

## Duas formas de projetar

O doc de MongoDB já cobre `ProductSummaryOutput` vs. `ProductDetailOutput`. Esta leva trouxe um terceiro tipo, que funciona num nível diferente:

```java
// domain/product/ProductRepository.java
@Query(value = "{'enabled': ?0}", fields = "{'name': 1}")
Page<ProductNameProjection> findAllByEnabled(Boolean enabled, Pageable pageable);
```

```java
public record ProductNameProjection(UUID id, String name) { }
```

| | Projeção na aplicação | Projeção no servidor |
|---|---|---|
| Onde recorta | Java, com ModelMapper | MongoDB, via `fields` |
| Trafega pela rede | documento **inteiro** | só `_id` e `name` |
| Exemplo | `ProductSummaryOutput` | `ProductNameProjection` |
| Quando usar | precisa de campo derivado/formatado | precisa de poucos campos crus |

O `fields = "{'name': 1}"` diz ao Mongo quais campos devolver (`1` inclui). O `_id` vem sempre, a menos que se peça `{'_id': 0}` explicitamente.

Para um autocomplete que só mostra o nome, trazer 15 campos por produto é desperdício puro de rede e de memória.

---

## Armadilhas

1. **Nome de campo é `String`.** Nada valida em tempo de compilação. Renomeou o campo no `Product`? A consulta continua compilando e para de achar.
2. **O nome é o do documento, não o do Java.** `Product.category` é gravado como `categoryId` por causa do `@Field`. A consulta usa `categoryId`.
3. **Dois criterias no mesmo campo estouram.** Encadeie no mesmo `where`.
4. **`count` depois de paginar mente.** Sempre antes.
5. **Divisão inteira no `totalPages`.** O cast para `double` não é decorativo.
6. **`$expr` e regex sem âncora ignoram índice.** Funcionam; não escalam.

---

## Pendências registradas

- [ ] **A ordenação do cliente é ignorada.** `ProductFilter.getSortByPropertyOrDefault()` e `getSortDirectionOrDefault()` retornam constante fixa (`ADDED_AT`, `ASC`) sem olhar o que foi bindado. Na prática `?sortByProperty=SALE_PRICE&sortDirection=DESC` não faz nada. Falta o fallback:
  ```java
  return getSortByProperty() != null ? getSortByProperty() : SortType.ADDED_AT;
  ```
- [ ] **`regularRegex` está declarado e nunca é usado** — só o `flexibleRegex` entra na consulta. Ou vira opção de busca exata no filtro, ou sai.
- [ ] **O `term` entra cru na regex**, sem `Pattern.quote`. Um termo com `(`, `[` ou `*` quebra a consulta; um termo construído de propósito (`(a+)+$`) é vetor de ReDoS, já que a avaliação roda no servidor do banco.
- [ ] **Nenhum índice foi criado** para os campos filtrados (`enabled`, `salePrice`, `addedAt`, `categoryId`). Com a coleção pequena não aparece; com volume, toda consulta é varredura.
- [ ] **Não há teste do `queryWith`.** Toda a lógica de montagem — inclusive o `if/else` dos intervalos — está sem cobertura. Um teste que só inspecione `query.getQueryObject()` já pegaria a maior parte dos erros, sem precisar de Mongo de pé.

---

## Checklist de revisão

- [ ] Sei explicar por que 9 filtros opcionais não viram 9 métodos de repositório
- [ ] Sei a diferença entre `Query` e `Criteria`, e que `addCriteria` sucessivos são `AND`
- [ ] Entendo por que o intervalo precisa ser encadeado no mesmo `where`
- [ ] Sei por que `hasDiscount` exige `$expr`, e o preço disso
- [ ] Sei por que `orOperator` e não três `addCriteria` na busca textual
- [ ] Sei por que o `count` vem antes do `skip`/`limit`
- [ ] Sei distinguir projeção no servidor (`fields`) de projeção na aplicação (ModelMapper)
- [ ] Reconheço que `null` no filtro significa "não filtra", não "filtra por null"

---

## Referências

- [Spring Data MongoDB — Query by Criteria](https://docs.spring.io/spring-data/mongodb/reference/mongodb/template-query-operations.html)
- [MongoDB — `$expr`](https://www.mongodb.com/docs/manual/reference/operator/query/expr/)
- [MongoDB — `$regex` e o uso de índices](https://www.mongodb.com/docs/manual/reference/operator/query/regex/)
- [`paginacao.md`](./paginacao.md) — o equivalente com JPA Criteria API no `ordering`
- [`product-catalog-mongo.md`](./product-catalog-mongo.md) — modelagem do agregado consultado aqui
- [`cqrs.md`](../01-arquitetura-design/cqrs.md) — por que a consulta não passa pelo agregado
