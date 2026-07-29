# MongoDB no product-catalog

> Como o agregado `Product` foi modelado com Spring Data MongoDB no microsserviço `product-catalog`.
> Cobre modelagem documental, relacionamento por referência, UUID como `_id`, conversores customizados, auditoria, lock otimista e projeções de leitura.

---

## Por que MongoDB neste serviço?

O AlgaShop usa **PostgreSQL** no `ordering` e no `billing` — domínios transacionais, com invariantes fortes e dinheiro envolvido. O `product-catalog` é diferente:

| Característica do catálogo | Consequência |
|---|---|
| Muito mais leitura que escrita | Otimizar para consulta compensa |
| Produto tem atributos que variam por tipo (roupa tem tamanho, eletrônico tem voltagem) | Schema flexível ajuda |
| Não participa de transação com dinheiro | Perder ACID multi-documento não dói |
| Cresce em volume mais rápido que os outros | Escala horizontal importa |

Isso não é "MongoDB é melhor". É **persistência poliglota**: cada serviço escolhe o banco que combina com o padrão de acesso dele. Como cada microsserviço tem banco próprio, essa escolha é local e não contamina os outros.

> Base conceitual de NoSQL, teorema CAP e famílias de banco: [`nosql-conceitos.md`](./nosql-conceitos.md).

---

## 1. Modelagem documental — `@Document`

```java
// domain/product/Product.java
@Document(collection = "products")
@Getter
@EqualsAndHashCode(onlyExplicitlyIncluded = true)
@NoArgsConstructor
public class Product {

    @Id
    @EqualsAndHashCode.Include
    private UUID id;

    private String name;
    private String brand;
    private String description;
    private Integer quantityInStock = 0;
    private Boolean enabled;
    private BigDecimal regularPrice;
    private BigDecimal salePrice;
    // ...
}
```

Comparando com o mundo JPA do `ordering`:

| JPA (ordering) | Spring Data MongoDB (product-catalog) |
|---|---|
| `@Entity` + `@Table(name = "orders")` | `@Document(collection = "products")` |
| `@Id` de `jakarta.persistence` | `@Id` de `org.springframework.data.annotation` |
| `@Column(...)` | `@Field(name = "...")` (só quando o nome difere) |
| `@ManyToOne` / `@OneToMany` | `@DocumentReference` ou documento embutido |
| Schema criado por migration (Flyway) | Sem schema — a coleção nasce no primeiro insert |

**Atenção ao import do `@Id`.** É `org.springframework.data.annotation.Id`, não `jakarta.persistence.Id`. Como o projeto costuma ter as duas dependências no classpath em outros serviços, a IDE sugere a errada com facilidade — e o sintoma é o Mongo gerar um `ObjectId` próprio e ignorar o seu campo.

O `@EqualsAndHashCode(onlyExplicitlyIncluded = true)` com `@EqualsAndHashCode.Include` só no `id` mantém a regra de DDD: **duas entidades são iguais quando têm a mesma identidade**, não quando têm os mesmos atributos.

---

## 2. Relacionamento por referência — `@DocumentReference`

A decisão de modelagem mais importante do documento. Em Mongo existem dois caminhos para representar "produto pertence a uma categoria":

**Opção A — embutir (embedded):**
```json
{ "_id": "...", "name": "Notebook", "category": { "name": "Eletrônicos", "enabled": true } }
```
Leitura em uma única ida ao banco. Mas renomear uma categoria exige atualizar **todos** os produtos dela.

**Opção B — referenciar (a escolhida):**
```java
// domain/product/Product.java
@DocumentReference
@Field(name = "categoryId")
private Category category;
```
```json
{ "_id": "...", "name": "Notebook", "categoryId": "0197f2c1-..." }
```

O documento guarda só o id; o Spring Data resolve o objeto `Category` quando você acessa o getter.

**Por que referência aqui:** categoria é uma entidade com ciclo de vida próprio (é criada, renomeada e desabilitada pela API de categorias, independente de produto). Duplicar o nome dela em milhares de produtos criaria um problema de consistência a cada rename. Categoria muda pouco, mas quando muda, muda para todo mundo.

**O custo:** cada leitura de produto que toca `getCategory()` dispara uma consulta extra na coleção `categories`. Em uma listagem de 20 produtos isso pode virar 20 consultas (o clássico **N+1**). Ponto de atenção real para quando a listagem for implementada.

> Regra prática: **embuta o que é lido junto e muda junto; referencie o que tem vida própria.**

---

## 3. UUID como `_id` — `UuidRepresentation.STANDARD`

O id não é sequencial nem `ObjectId`, é um **UUID v7** gerado na aplicação:

```java
// domain/util/IdGenerator.java
private static final TimeBasedEpochRandomGenerator timeBasedEpochRandomGenerator
        = Generators.timeBasedEpochRandomGenerator();

public static UUID generateTimeBasedUUID() {
    return timeBasedEpochRandomGenerator.generate();
}
```

UUID v7 é **ordenável por tempo** — os primeiros bits carregam o timestamp. Isso evita o problema clássico do UUID v4 aleatório, que espalha as escritas pelo índice e fragmenta a B-tree.

O problema: o driver do Mongo tem **várias representações binárias** de UUID por razões históricas (legado Java, legado C#, legado Python). Se você não fixar uma, corre o risco de gravar em um formato e ler em outro — e o id "some".

```java
// infrastructure/persistence/MongoConfig.java
@Bean
public MongoClientSettingsBuilderCustomizer uuidCustomizer() {
    return builder -> builder.uuidRepresentation(UuidRepresentation.STANDARD);
}
```

`STANDARD` é o formato correto e interoperável (RFC 4122, subtipo BSON `0x04`). Sempre fixe isso explicitamente.

---

## 4. Conversores customizados — `OffsetDateTime`

O BSON **não tem** um tipo com offset de fuso. Ele tem `Date`, que é um instante em UTC. Como o domínio usa `OffsetDateTime`, é preciso ensinar a conversão nos dois sentidos:

```java
// infrastructure/persistence/MongoConfig.java
@Bean
public MongoCustomConversions customConversions() {
    return new MongoCustomConversions(
            List.of(new OffsetDatetimeReadConverter(), new OffsetDatetimeWriteConverter())
    );
}

// Mongo -> domínio
public static class OffsetDatetimeReadConverter implements Converter<Date, OffsetDateTime> {
    @Override
    public @Nullable OffsetDateTime convert(Date source) {
        return source.toInstant().atZone(ZoneId.systemDefault()).toOffsetDateTime();
    }
}

// domínio -> Mongo
public static class OffsetDatetimeWriteConverter implements Converter<OffsetDateTime, Date> {
    @Override
    public @Nullable Date convert(OffsetDateTime source) {
        return Date.from(source.toInstant());
    }
}
```

**Consequência que vale entender:** a escrita é fiel (o instante é preservado), mas a leitura reconstrói o offset a partir do `ZoneId.systemDefault()` — ou seja, do fuso de quem está lendo. Se você gravar `10:00-03:00` e ler em um servidor UTC, recebe `13:00Z`. É o **mesmo instante**, mas o offset original não volta. Para catálogo isso é irrelevante; para um domínio onde "o fuso em que o usuário digitou" importa, seria preciso guardar o offset em um campo separado.

---

## 5. Auditoria — quem criou, quando, e por quem foi alterado

Em vez de setar `createdAt` na mão dentro do construtor, a responsabilidade passa para o Spring Data:

```java
// infrastructure/persistence/SpringDataAuditingConfig.java
@Configuration
@EnableMongoAuditing(
    dateTimeProviderRef = "auditingDateTimeProvider",
    auditorAwareRef = "auditorProvider"
)
public class SpringDataAuditingConfig {

    @Bean
    public DateTimeProvider auditingDateTimeProvider() {
        return () -> Optional.of(OffsetDateTime.now().truncatedTo(ChronoUnit.MILLIS));
    }

    @Bean
    public AuditorAware<UUID> auditorProvider() {
        return () -> Optional.of(UUID.randomUUID());
    }
}
```

E as anotações no agregado:

```java
@CreatedDate       private OffsetDateTime addedAt;
@LastModifiedDate  private OffsetDateTime updatedAt;
@CreatedBy         private UUID createdByUserId;
@LastModifiedBy    private UUID lastModifiedByUserId;
```

**Por que `truncatedTo(ChronoUnit.MILLIS)`:** o `OffsetDateTime` do Java tem precisão de **nanossegundos**; o `Date` do BSON tem precisão de **milissegundos**. Sem o truncamento, o objeto em memória e o objeto lido do banco nunca são iguais — e testes que comparam datas falham com uma diferença invisível a olho nu. Truncar na origem alinha os dois mundos.

Repare que o construtor de `Category` **perdeu** a linha `this.createdAt = OffsetDateTime.now()`. Isso é intencional: com auditoria ativa, quem preenche é a infraestrutura, no momento do `save()`.

> ⚠️ **Pendência conhecida:** `auditorProvider()` devolve `UUID.randomUUID()` — um placeholder. Cada gravação registra um "usuário" diferente e inventado. Quando houver autenticação, isso vira algo como ler o id do usuário do `SecurityContextHolder`.

---

## 6. Lock otimista — `@Version`

```java
@Version
private Long version;
```

O campo é gerenciado pelo Spring Data. Na prática:

1. Você lê o produto — vem com `version: 3`.
2. Outra requisição lê o mesmo produto — também `version: 3`.
3. A primeira salva → o update roda com filtro `{_id: ..., version: 3}` e grava `version: 4`.
4. A segunda salva → o filtro `{_id: ..., version: 3}` **não casa com nada**, e o Spring lança `OptimisticLockingFailureException`.

Sem isso, a segunda escrita simplesmente sobrescreveria a primeira em silêncio (*lost update*). Chama-se "otimista" porque não trava nada: assume que conflito é raro e só detecta quando acontece.

---

## 7. Regra de negócio dentro do agregado

O `Product` **não** é um saco de getters e setters. Ele protege os próprios invariantes:

```java
public void setRegularPrice(BigDecimal regularPrice) {
    Objects.requireNonNull(regularPrice);

    if (regularPrice.signum() == -1) {          // preço negativo não existe
        throw new IllegalArgumentException();
    }

    if (this.salePrice == null) {
        this.salePrice = regularPrice;           // sem promoção: os dois são iguais
    } else if (regularPrice.compareTo(this.salePrice) < 0) {
        throw new DomainException("Sale price cannot be greater than regular price");
    }

    this.regularPrice = regularPrice;
    this.calculateDiscountPercentage();          // derivado é recalculado, nunca setado de fora
}
```

Três lições concretas:

1. **`compareTo`, nunca `equals`, com `BigDecimal`.** `new BigDecimal("10.0").equals(new BigDecimal("10.00"))` é `false` — o `equals` compara também a escala. `compareTo` compara valor.
2. **O invariante mora no agregado.** "Preço promocional não pode ser maior que o preço normal" é verificado no setter. Não existe caminho — nem application service, nem controller — que consiga criar um produto violando isso.
3. **Campo derivado é calculado, não recebido.** `discountPercentageRounded` não tem setter público:

```java
private void calculateDiscountPercentage() {
    if (regularPrice == null || salePrice == null || regularPrice.signum() == 0) {
        discountPercentageRounded = 0;
        return;
    }

    discountPercentageRounded = BigDecimal.ONE
            .subtract(salePrice.divide(regularPrice, 4, RoundingMode.HALF_DOWN))
            .multiply(BigDecimal.valueOf(100))
            .setScale(0, RoundingMode.HALF_DOWN)
            .intValue();
}
```

O `divide` com escala e `RoundingMode` explícitos é obrigatório: `BigDecimal.divide` sem esses argumentos lança `ArithmeticException` em dízima periódica (ex.: `1/3`). O guard `regularPrice.signum() == 0` evita a divisão por zero.

Métodos de intenção fecham o desenho — `disable()`, `enable()`, `isInStock()`, `getHasDiscount()` — em vez de deixar quem chama manipular booleanos soltos.

---

## 8. Projeções de leitura — Summary vs. Detail

Antes existia um único `ProductDetailOutput` servindo tanto a listagem quanto o detalhe. Agora são dois:

```java
// A assinatura da listagem mudou de Detail para Summary
public interface ProductQueryService {
    ProductDetailOutput findById(UUID productId);
    PageModel<ProductSummaryOutput> filter(ProductFilter filter);   // <-- Summary
}
```

| | `ProductSummaryOutput` (lista) | `ProductDetailOutput` (detalhe) |
|---|---|---|
| Descrição | `shortDescription` (abreviada em 15 chars) | `description` completa |
| Uso | grid/listagem de produtos | página do produto |

Por que separar: uma listagem de 20 produtos não deve trafegar 20 descrições completas. O payload cresce sem que ninguém leia aquilo na tela. É a mesma ideia de projeção já usada no `ordering` — ver [`paginacao.md`](./paginacao.md) e [`cqrs.md`](../01-arquitetura-design/cqrs.md).

### Um terceiro tipo: projeção no servidor

Os dois DTOs acima recortam o dado **na aplicação** — o documento inteiro sai do banco e o ModelMapper descarta o que sobra. Dá para recortar antes, no próprio Mongo:

```java
// domain/product/ProductRepository.java
@Query(value = "{'enabled': ?0}", fields = "{'name': 1}")
Page<ProductNameProjection> findAllByEnabled(Boolean enabled, Pageable pageable);
```

```java
public record ProductNameProjection(UUID id, String name) { }
```

O `fields` diz quais campos devolver — só `_id` e `name` trafegam pela rede. Para um autocomplete, é a diferença entre trazer 2 campos e trazer 15.

> Quando usar cada um e o resto da mecânica de consulta: [`consultas-mongo-criteria.md`](./consultas-mongo-criteria.md).

> ℹ️ O `PageModel` mudou de pacote nesta etapa: `application` → `application.util`, junto com o `PageFilter` e o `SortablePageFilter` que nasceram com o filtro dinâmico.

---

## 9. ModelMapper com conversores customizados

O mapeamento entidade → DTO não é só cópia de campo. Dois campos do DTO **não existem** no `Product` e são derivados no momento da conversão:

```java
// infrastructure/util/mapper/ModelMapperConfig.java
private final Converter<String, String> fromStringToSlugConverter = ctx ->
        Slugfier.slugify(ctx.getSource());

private final Converter<String, String> fromStringToShortStringConverter = ctx ->
        StringUtils.abbreviate(ctx.getSource(), 15);

private void configuration(ModelMapper modelMapper) {
    modelMapper.getConfiguration()
            .setSourceNamingConvention(NamingConventions.NONE)
            .setDestinationNamingConvention(NamingConventions.NONE)
            .setMatchingStrategy(MatchingStrategies.STRICT);

    modelMapper.createTypeMap(Product.class, ProductDetailOutput.class)
            .addMappings(mappings -> mappings
                    .using(fromStringToSlugConverter)
                    .map(Product::getName, ProductDetailOutput::setSlug));

    modelMapper.createTypeMap(Product.class, ProductSummaryOutput.class)
            .addMappings(mappings -> {
                mappings.using(fromStringToSlugConverter)
                        .map(Product::getName, ProductSummaryOutput::setSlug);
                mappings.using(fromStringToShortStringConverter)
                        .map(Product::getDescription, ProductSummaryOutput::setShortDescription);
            });
}
```

**`MatchingStrategies.STRICT`** é a decisão que evita a maior armadilha do ModelMapper: nas estratégias `STANDARD` e `LOOSE` a biblioteca tenta adivinhar correspondências por similaridade de nome, e mapeia campos errados sem avisar. `STRICT` exige nome igual — o que não casar precisa de mapeamento explícito, como os dois acima.

E o slug em si:

```java
// infrastructure/util/Slugfier.java
public static String slugify(String text) {
    if (text == null) return null;
    String nowhitespace = WHITESPACE.matcher(text).replaceAll("-");
    String normalized = Normalizer.normalize(nowhitespace, Normalizer.Form.NFD);
    String slug = NONLATIN.matcher(normalized).replaceAll("");
    return slug.toLowerCase(Locale.ENGLISH);
}
```

O truque é o `Normalizer.Form.NFD`: ele **decompõe** `"ç"` em `"c"` + cedilha combinante, e `"á"` em `"a"` + acento combinante. O regex `[^\w-]` seguinte remove os acentos soltos e sobra o ASCII. Resultado: `"Notebook Ação Pro"` → `"notebook-acao-pro"`.

`Locale.ENGLISH` no `toLowerCase` também é proposital — evita o famoso problema do turco, onde `"I".toLowerCase()` produz `"ı"` (i sem ponto) no locale `tr`.

---

## 10. Configuração — atenção ao Spring Boot 4

```yaml
# src/main/resources/application.yml
server.port: 8083

spring:
  application:
    name: product-catalog
  mongodb:
    uri: mongodb://root:algashop@localhost:27017/product_catalog?authSource=admin
```

> ⚠️ **Armadilha de versão.** O projeto usa **Spring Boot 4.0**, onde a autoconfiguração do MongoDB saiu de `spring-boot-autoconfigure` para o módulo `spring-boot-mongodb` (pacote `org.springframework.boot.mongodb.autoconfigure`) e a propriedade passou de `spring.data.mongodb.uri` para **`spring.mongodb.uri`**.
>
> Praticamente todo tutorial e resposta de fórum ainda usa `spring.data.mongodb.*`. Se copiar de lá, a propriedade é silenciosamente ignorada e a aplicação conecta no default (`localhost:27017/test`, sem autenticação) — o sintoma é "conecta, mas a coleção está sempre vazia".

O `authSource=admin` é necessário porque o usuário `root` foi criado no banco `admin` (ver `MONGO_INITDB_ROOT_USERNAME` no compose), enquanto os dados vivem em `product_catalog`. Sem isso, o Mongo procura as credenciais no banco errado e recusa a conexão.

Para subir o Mongo local: [`../04-infraestrutura/ambiente-local.md`](../04-infraestrutura/ambiente-local.md).

---

## Pendências registradas

Coisas que ficaram conscientemente pela metade nesta etapa:

- ✅ ~~`ProductQueryServiceImpl.filter()` ainda retorna `null`~~ — **implementado** com `Query` + `Criteria` e paginação manual; ver [`consultas-mongo-criteria.md`](./consultas-mongo-criteria.md).
- [ ] Não há testes cobrindo o agregado `Product` (nem os invariantes de preço, nem a persistência). O `ProductRepositoryIT` que surgiu depois exercita só a projeção do repositório, e sem asserção — apenas loga o resultado.
- [ ] `auditorProvider()` devolve `UUID.randomUUID()` até existir autenticação.
- [ ] `quantityInStock` tem `setQuantityInStock` **privado** e nenhum método público de entrada/saída de estoque. Produto criado **pela API** fica travado em `0`, e `isInStock()` sempre retorna `false`. Os documentos carregados pelo [`DataLoader`](../04-infraestrutura/carga-de-dados-mongo.md) escapam disso porque são inseridos crus, direto na coleção — por isso o filtro `?inStock=true` funciona nos dados de teste e não funcionaria num produto cadastrado pelo endpoint.
- [ ] N+1 latente no `@DocumentReference`: a listagem já está implementada, e cada `ProductSummaryOutput` que exponha o nome da categoria dispara uma leitura extra.

---

## Checklist de revisão

- [ ] `@Id` importado de `org.springframework.data.annotation`, não de `jakarta.persistence`
- [ ] `UuidRepresentation.STANDARD` fixado explicitamente
- [ ] Conversores de `OffsetDateTime` registrados nos dois sentidos
- [ ] `DateTimeProvider` truncando em millis para bater com a precisão do BSON
- [ ] `@Version` presente nos agregados que sofrem escrita concorrente
- [ ] `MatchingStrategies.STRICT` no ModelMapper
- [ ] `compareTo` (não `equals`) ao comparar `BigDecimal`
- [ ] `divide` de `BigDecimal` sempre com escala e `RoundingMode`
- [ ] Propriedade `spring.mongodb.uri` (Boot 4), não `spring.data.mongodb.uri`

---

## Referências

- [Spring Data MongoDB — Reference](https://docs.spring.io/spring-data/mongodb/reference/)
- [MongoDB — Data Model Design (embed vs. reference)](https://www.mongodb.com/docs/manual/core/data-model-design/)
- [RFC 9562 — UUID v7](https://www.rfc-editor.org/rfc/rfc9562)
- [`consultas-mongo-criteria.md`](./consultas-mongo-criteria.md) — como consultar esse modelo
- [`carga-de-dados-mongo.md`](../04-infraestrutura/carga-de-dados-mongo.md) — como popular a coleção
