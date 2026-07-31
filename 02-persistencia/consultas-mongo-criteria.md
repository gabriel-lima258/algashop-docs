# Consultas dinâmicas no MongoDB com `Criteria`

> Como o `product-catalog` monta um filtro de listagem com 9 parâmetros opcionais sem escrever 512 consultas.
> Código real: `infrastructure/persistence/product/ProductQueryServiceImpl.java`, `.../category/CategoryQueryServiceImpl.java`.

> ℹ️ **Este é o jeito 1 de consultar.** A listagem de produtos migrou depois para um **aggregation pipeline** — [`agregacoes-mongo.md`](./agregacoes-mongo.md) — para juntar a coleção de categorias e calcular campos derivados no servidor.
>
> Nada aqui ficou obsoleto: o `findById` e o `CategoryQueryServiceImpl` continuam neste jeito, e o **`Criteria` desta página é o mesmo que alimenta o `$match` do pipeline**. O que muda é como ele é montado e o que vem depois. A seção [Quando usar cada um](#quando-usar-cada-um) fecha a comparação.

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

Por que isso importa: se `sortByProperty` fosse `String`, o cliente poderia mandar `?sortByProperty=senhaDoAdmin` e o Mongo tentaria ordenar por um campo qualquer. Com enum, **o conjunto de campos ordenáveis é fechado** — valor fora da lista morre na conversão, com 400, e nunca chega ao serviço. É a mesma ideia de uma allowlist.

> Repare que a constante tem **dois nomes**: `SALE_PRICE`, o do Java, e `salePrice`, o do campo no documento. Essa dualidade parece inofensiva e é a origem de um bug real — ver [Conversão: o enum que o cliente não sabe escrever](#conversão-o-enum-que-o-cliente-não-sabe-escrever).

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

## Conversão: o enum que o cliente não sabe escrever

O binding acima só funciona se o Spring souber transformar **texto** no tipo de cada campo. Para `Integer` e `BigDecimal` isso é automático; para os enums de ordenação, não era — e o sintoma foi um 400 numa requisição perfeitamente razoável:

```
GET /api/v1/products?sortByProperty=salePrice&sortDirection=desc
```

```json
{
  "status": 400,
  "title": "Invalid fields",
  "fields": {
    "sortDirection": "Failed to convert property value of type java.lang.String to required type org.springframework.data.domain.Sort$Direction ... for value [desc]",
    "sortByProperty": "Failed to convert ... to required type ...ProductFilter$SortType for value [salePrice]"
  }
}
```

Duas falhas independentes, e as duas com a mesma raiz: **o conversor padrão do Spring para enum é um `valueOf`**.

| Enviado | O conversor padrão espera | Resultado |
|---|---|---|
| `desc` | `DESC` | 400 — `valueOf` diferencia maiúscula |
| `salePrice` | `SALE_PRICE` | 400 — é o nome do **campo**, não o da **constante** |

O segundo caso é o mais instrutivo. O enum tem dois nomes, e o cliente escreve o **errado por um bom motivo**: `salePrice` é o que aparece no JSON de resposta e o que o contrato OpenAPI do próprio serviço publica (`docs/openapi/product-catalog.yml`, `example: "name"`). Quem consome a API não tem como adivinhar que existe um segundo nome, em `SCREAMING_SNAKE_CASE`, escondido no código Java.

**A lição:** *type-safe na aplicação não é o mesmo que amigável no HTTP.* O enum resolve o problema de segurança (a allowlist) e cria um de tradução na fronteira. A fronteira precisa ser escrita à mão.

### A tradução, num arquivo

`addFormatters` é o gancho de conversão da camada web. Registrar ali é o passo fácil de esquecer: sem ele, conversor nenhum entra em jogo.

```java
// infrastructure/web/WebConfig.java
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addFormatters(FormatterRegistry registry) {
        registry.addConverter(String.class, Sort.Direction.class,
                source -> Sort.Direction.fromString(source.trim()));

        registry.addConverter(String.class, ProductFilter.SortType.class,
                source -> resolve(ProductFilter.SortType.values(), source,
                        ProductFilter.SortType::getPropertyName));

        registry.addConverter(String.class, CategoryFilter.SortType.class,
                source -> resolve(CategoryFilter.SortType.values(), source,
                        CategoryFilter.SortType::getPropertyName));
    }

    private <T extends Enum<T>> T resolve(T[] values, String source, Function<T, String> propertyName) {
        String value = source.trim();
        return Arrays.stream(values)
                .filter(constant -> value.equalsIgnoreCase(propertyName.apply(constant))
                        || value.equalsIgnoreCase(constant.name()))
                .findFirst()
                .orElseThrow(() -> new IllegalArgumentException(...));
    }
}
```

Para a direção não há lógica a escrever: `Sort.Direction.fromString`, do Spring Data, já normaliza a caixa e lança com mensagem legível. Para as propriedades, o `resolve` aceita os dois nomes da constante, ignorando caixa.

O overload `addConverter(Class<S>, Class<T>, Converter)` aceita lambda — é o que dispensa uma classe por conversor.

> #### Nota de estudo: por que **não** um `ConverterFactory`
>
> Existe uma abstração do Spring exatamente para "converter para uma família de tipos": o `ConverterFactory<S, R>`, que produz um conversor por tipo de destino. Aqui ele encaixaria — o alvo são vários enums de ordenação.
>
> A primeira versão desta correção fez assim: uma interface `SortableProperty` (para dar ao factory um alvo restrito, já que um `ConverterFactory<String, Enum>` sequestraria a conversão de enum da aplicação inteira), os dois enums implementando, o genérico do `SortablePageFilter` apertado para `<T extends Enum<T> & SortableProperty>`, e o factory. Oito arquivos.
>
> **Não valeu a pena.** O `ConverterFactory` se paga quando a família de tipos é aberta ou grande; aqui são **dois** enums, e cada um custa uma linha de registro. O argumento de que a interface era necessária para apertar o genérico também não se sustentava: `<T extends Enum<T>>` já barra `String`, que era o ponto, e não custa arquivo nenhum.
>
> Fica o critério, que é o que vale levar: **`Converter` quando os destinos são poucos e conhecidos; `ConverterFactory` quando são muitos ou imprevisíveis.** O preço de escolher o primeiro está declarado nas pendências — filtro ordenável novo precisa lembrar de entrar no `WebConfig`.

### A mensagem de erro também é parte do contrato

Valor inválido de verdade (`?sortByProperty=hackme`) **deve** continuar dando 400 — é a allowlist funcionando. O que não deve é a resposta pública conter `com.algaworks...ProductFilter$SortType`.

O caminho óbvio seria um `messages.properties` com os códigos `typeMismatch`, que o `ApiExceptionHandler` já consulta. Funciona, e tem um defeito: a lista de valores aceitos ficaria escrita à mão num arquivo, longe do enum. Valor novo no `SortType` e a mensagem passa a mentir, sem nada acusar.

A saída é ir na **causa raiz** do erro de conversão, onde já existe uma mensagem boa — a que o próprio conversor lançou:

```java
// presentation/ApiExceptionHandler.java
private String messageOf(ObjectError objectError) {
    if (objectError.contains(TypeMismatchException.class)) {
        return objectError.unwrap(TypeMismatchException.class)
                .getMostSpecificCause().getMessage();
    }
    return messageSource.getMessage(objectError, LocaleContextHolder.getLocale());
}
```

Dois tipos de erro chegam no mesmo handler e agora são tratados conforme a natureza de cada um:

| Origem | Como a mensagem é resolvida |
|---|---|
| **Conversão** (texto que não vira o tipo do campo) | causa raiz — a exceção do conversor |
| **Validação** (Bean Validation) | `MessageSource`, como sempre |

Resultado:

```json
"fields": { "sortByProperty": "Invalid value 'hackme'; must be one of: addedAt, salePrice" }
```

A lista sai do `values()` do enum, então **nunca desatualiza** — e nenhum arquivo de mensagens foi criado.

> `ObjectError.contains(Class)` e `unwrap(Class)` existem desde o Spring 5.3, e é o que dá acesso à exceção original por trás do erro de binding.

### O que ficou coberto por teste

Um teste só, e é o que precisa existir: o `SortParameterBindingTest`, uma fatia `@WebMvcTest` que exercita a cadeia inteira — `WebConfig` → binding → `ApiExceptionHandler`.

A escolha de testar pela API, e não classe por classe, foi o que permitiu enxugar o código depois sem tocar em nenhuma asserção: as quatro chamadas continuam sendo as mesmas quatro chamadas, com os mesmos resultados. Teste amarrado às classes internas teria virado trabalho de refatoração.

---

## Montagem incremental

O coração do filtro dinâmico cabe em quatro linhas de padrão:

```java
private Query queryWith(CategoryFilter filter) {
    Query query = new Query();

    if (filter.getEnabled() != null) {
        query.addCriteria(Criteria.where("enabled").is(filter.getEnabled()));
    }
    // ... um bloco desses por filtro

    return query;
}
```

`null` significa **"não filtra por isso"** — não significa "filtra por null". É a diferença entre "tanto faz se está ativo" e "traga os que têm `enabled: null`".

> O produto usa hoje uma variação disso: o método se chama `buildCriteria`, acumula os criterias numa `List<Criteria>` e devolve `Optional.of(new Criteria().andOperator(criterias))` em vez de empilhar na `Query`. O motivo é o destino — um estágio `$match` recebe **um** criteria, não uma coleção deles. A lógica de cada filtro é idêntica; ver [`agregacoes-mongo.md`](./agregacoes-mongo.md#match--o-criteria-reaproveitado).

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
| `term` | `TextCriteria.forDefaultLanguage().matching(x)` | `{ $text: { $search: "x" } }` |

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
>
> Agora que os demais campos ganharam índice ([`indices-mongo.md`](./indices-mongo.md)), esse custo ficou mais visível por contraste: é o único filtro que continua obrigando o Mongo a olhar documento por documento.

> 🔧 **Mudou na migração para aggregation.** O `AggregationExpressionCriteria` implementa `CriteriaDefinition` mas **não** `Criteria`, e o `andOperator` só aceita `Criteria` — então no pipeline o mesmo `$expr` é escrito como criteria comum:
>
> ```java
> criterias.add(Criteria.where("$expr").is(discountExpression.toDocument()));
> ```
>
> O BSON gerado é o mesmo; muda só o caminho até ele. Detalhe em [`agregacoes-mongo.md`](./agregacoes-mongo.md#o-detalhe-de-tipos-que-morde-expr).

---

## Busca textual: de regex para `$text`

Esta busca nasceu como um `$or` de três expressões regulares e foi **substituída** por índice de texto. Vale guardar as duas versões, porque a troca resume um trade-off que reaparece sempre.

**Antes** — três regex agrupadas com `$or`:

```java
private static final String flexibleRegex = "(?i)%s";

String regexExpression = String.format(flexibleRegex, filter.getTerm());
query.addCriteria(
        new Criteria().orOperator(
                Criteria.where("name").regex(regexExpression),
                Criteria.where("brand").regex(regexExpression),
                Criteria.where("description").regex(regexExpression)
        )
);
```

O `orOperator` agrupava os três num único criteria com `$or`. Se fossem três `addCriteria`, virariam `AND` — e o produto precisaria ter o termo nos três campos ao mesmo tempo. (O `(?i)` ligava o modo case-insensitive dentro da própria regex, e o `%s` era do `String.format`, não do Mongo.)

**Agora** — um `$text` sobre o índice de texto da coleção:

```java
if (StringUtils.isNotBlank(filter.getTerm())) {
    query.addCriteria(TextCriteria.forDefaultLanguage().matching(filter.getTerm()));
}
```

O que a troca custou e o que rendeu:

| | `$or` de regex | `$text` |
|---|---|---|
| Usa índice | não — varre a coleção | sim |
| `?term=note` acha "Notebook" | sim | **não** — casa palavra inteira |
| Ordenação por relevância | não existe | nativa (campo `score`) |
| Termo malicioso | ReDoS avaliado no servidor | inofensivo |
| Campos cobertos | `name`, `brand`, `description` | `name`, `description` |

Duas consequências que não são detalhe: **busca enquanto o usuário digita deixou de funcionar**, e **marca saiu da busca**.

Quando há termo, a ordenação pedida pelo cliente é ignorada em favor da relevância:

```java
private Sort sortWith(ProductFilter filter) {
    if (StringUtils.isNotBlank(filter.getTerm())) {
        return Sort.by("score");   // vira { score: { $meta: "textScore" } }
    }
    return Sort.by(filter.getSortDirectionOrDefault(),
            filter.getSortByPropertyOrDefault().getPropertyName());
}
```

O detalhe de como `"score"` vira `$meta`, os pesos, o stemming e o índice de texto em si estão em **[`indices-mongo.md`](./indices-mongo.md)**.

---

## O mesmo padrão, segunda aplicação

O `CategoryQueryServiceImpl` foi implementado nesta etapa (antes o `filter()` devolvia `null`) e é o mesmo esqueleto, campo a campo: `queryWith` → `count` → `sortWith` → `with(pageRequest)` → `PageModel`. A diferença é só o tamanho do `queryWith`, que tem dois filtros em vez de nove.

E hoje ele é o **melhor exemplo vivo deste jeito** no projeto: enquanto o produto migrou para pipeline, a categoria não precisou — não junta coleção nenhuma nem calcula campo derivado, então o `find()` continua sendo a resposta certa.

É a prova de que a hierarquia `PageFilter` → `SortablePageFilter<T>` → `Filter` se paga: o `CategoryFilter` nasceu com paginação e ordenação type-safe funcionando, sem escrever nada disso de novo.

Um contraste vale registrar: a categoria continua buscando por nome com regex não ancorada, e a coleção `categories` não tem índice nenhum — o oposto do que o produto acabou de adotar. Com poucas categorias não incomoda; é dívida consciente, não descuido.

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

## Três formas de projetar

O doc de MongoDB já cobre `ProductSummaryOutput` vs. `ProductDetailOutput`. Esta leva trouxe um terceiro tipo, que funciona num nível diferente:

```java
// domain/product/ProductRepository.java
@Query(value = "{'enabled': ?0}", fields = "{'name': 1}")
Page<ProductNameProjection> findAllByEnabled(Boolean enabled, Pageable pageable);
```

```java
public record ProductNameProjection(UUID id, String name) { }
```

E o aggregation trouxe a terceira:

| | Na aplicação | No servidor, com `fields` | No servidor, com `$project` |
|---|---|---|---|
| Onde recorta | Java, com ModelMapper | MongoDB | MongoDB |
| Trafega pela rede | documento **inteiro** | só os campos pedidos | só os campos pedidos |
| Calcula campo derivado | sim, em Java | **não** — só inclui/exclui | sim, com operador do Mongo |
| Junta outra coleção | não | não | sim, com `$lookup` |
| Exemplo | `ProductDetailOutput` | `ProductNameProjection` | `ProductSummaryOutput` |
| Quando usar | derivar/formatar é barato em Java | poucos campos crus | derivar e juntar no banco compensa |

O `fields = "{'name': 1}"` diz ao Mongo quais campos devolver (`1` inclui). O `_id` vem sempre, a menos que se peça `{'_id': 0}` explicitamente.

Para um autocomplete que só mostra o nome, trazer 15 campos por produto é desperdício puro de rede e de memória.

A terceira coluna é o assunto de [`agregacoes-mongo.md`](./agregacoes-mongo.md#project-derivar-no-servidor): foi o `$project` que permitiu ao `ProductSummaryOutput` sair do ModelMapper e virar destino direto da consulta.

---

## Armadilhas

1. **Nome de campo é `String`.** Nada valida em tempo de compilação. Renomeou o campo no `Product`? A consulta continua compilando e para de achar.
2. **O nome é o do documento, não o do Java.** `Product.category` é gravado como `categoryId` por causa do `@Field`. A consulta usa `categoryId`.
3. **Dois criterias no mesmo campo estouram.** Encadeie no mesmo `where`.
4. **`count` depois de paginar mente.** Sempre antes.
5. **Divisão inteira no `totalPages`.** O cast para `double` não é decorativo.
6. **`$expr` e regex sem âncora ignoram índice.** Funcionam; não escalam.
7. **Enum type-safe não conversa sozinho com a query string.** O conversor padrão é `valueOf`: sensível a caixa e cego para qualquer nome que não seja o da constante. A fronteira HTTP precisa de tradução explícita.
8. **A mensagem padrão de erro de conversão vaza nome de classe Java** na resposta pública. A causa raiz costuma trazer texto melhor que a do framework.

---

## Pendências registradas

- [x] ~~**A ordenação do cliente é ignorada.**~~ Corrigido em duas etapas: primeiro o `ProductFilter` passou a cair no default só quando o campo vem `null`; depois os conversores fizeram a query string chegar até lá. Hoje `?sortByProperty=salePrice&sortDirection=desc` funciona — e `SALE_PRICE`/`DESC` também.
- [x] ~~**Nenhum índice foi criado** para os campos filtrados.~~ Resolvido — ver [`indices-mongo.md`](./indices-mongo.md).
- [x] ~~**`regularRegex` está declarado e nunca é usado.**~~ As duas constantes de regex saíram junto com a troca para `$text`.
- [ ] **O nome da categoria entra cru na regex.** A pendência do `Pattern.quote` migrou do produto (que não usa mais regex) para o `CategoryQueryServiceImpl`: `Criteria.where("name").regex(filter.getName().trim(), "i")`. Um termo com `(`, `[` ou `*` quebra a consulta; um termo construído de propósito (`(a+)+$`) é vetor de ReDoS, já que a avaliação roda no servidor do banco.
- [ ] **A coleção `categories` não tem índice**, e a busca por nome é regex sem âncora — varredura garantida. Hoje é irrelevante pelo volume.
- [ ] **O registro de cada enum ordenável é manual no `WebConfig`.** Filtro novo que esqueça a linha reproduz o 400 do `sortByProperty`, e silenciosamente — nada no código acusa a falta. É o preço declarado de não usar um `ConverterFactory`; o `SortParameterBindingTest` só cobre os enums que já existem.
- [~] **Cobertura parcial da montagem.** O `ProductQueryServiceImplTest` cobre o `buildCriteria` na parte do `$expr` — inspecionando `query.getQueryObject()` com `ArgumentCaptor`, sem Mongo de pé, exatamente a abordagem barata que faltava. Falta estender ao `if/else` dos intervalos e ao `queryWith` das categorias.

---

## Quando usar cada um

Os dois jeitos convivem no `product-catalog`, e a escolha não é de gosto:

| | `Query` + `Criteria` (este doc) | [Aggregation pipeline](./agregacoes-mongo.md) |
|---|---|---|
| Complexidade de escrita | baixa | alta |
| Junta outra coleção | não | `$lookup` |
| Campo derivado | em Java, depois da consulta | `$project`, no banco |
| Projeção | `fields` ou ModelMapper | `$project` tipado direto no DTO |
| Uso de índice | direto, na consulta inteira | só no primeiro `$match` |
| Ordem de execução | o Mongo decide | **você** decide, estágio a estágio |
| Onde vive hoje | `findById`, `CategoryQueryServiceImpl` | listagem de produtos |

**Fique neste jeito** enquanto a consulta for sobre uma coleção só e os campos derivados forem baratos de calcular em Java. É menos código, mais legível, e o índice serve a consulta inteira.

**Vá para o pipeline** quando precisar juntar coleções, agrupar/somar, ou quando trazer o documento inteiro para recortar em Java virar desperdício mensurável — que foi exatamente o caso da listagem de produtos, com o N+1 do `@DocumentReference`.

---

## Checklist de revisão

- [ ] Sei explicar por que 9 filtros opcionais não viram 9 métodos de repositório
- [ ] Sei a diferença entre `Query` e `Criteria`, e que `addCriteria` sucessivos são `AND`
- [ ] Entendo por que o intervalo precisa ser encadeado no mesmo `where`
- [ ] Sei por que `hasDiscount` exige `$expr`, e o preço disso
- [ ] Sei o que se ganhou e o que se perdeu ao trocar o `$or` de regex por `$text`
- [ ] Sei por que o `count` vem antes do `skip`/`limit`
- [ ] Sei distinguir as três formas de projetar: ModelMapper, `fields` e `$project`
- [ ] Reconheço que `null` no filtro significa "não filtra", não "filtra por null"
- [ ] Sei decidir entre este jeito e o aggregation pipeline
- [ ] Sei por que um enum precisa de `Converter` para vir da query string, e quando um `ConverterFactory` se paga
- [ ] Sei tirar a mensagem de um erro de conversão da causa raiz, em vez de duplicar a lista de valores num arquivo

---

## Referências

- [Spring Data MongoDB — Query by Criteria](https://docs.spring.io/spring-data/mongodb/reference/mongodb/template-query-operations.html)
- [MongoDB — `$expr`](https://www.mongodb.com/docs/manual/reference/operator/query/expr/)
- [MongoDB — `$regex` e o uso de índices](https://www.mongodb.com/docs/manual/reference/operator/query/regex/)
- [`agregacoes-mongo.md`](./agregacoes-mongo.md) — o jeito 2, com aggregation pipeline
- [`indices-mongo.md`](./indices-mongo.md) — os índices que servem estas consultas, e o `$text` da busca por termo
- [`paginacao.md`](./paginacao.md) — o equivalente com JPA Criteria API no `ordering`
- [`product-catalog-mongo.md`](./product-catalog-mongo.md) — modelagem do agregado consultado aqui
- [`cqrs.md`](../01-arquitetura-design/cqrs.md) — por que a consulta não passa pelo agregado
