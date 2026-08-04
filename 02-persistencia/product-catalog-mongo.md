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
| `@ManyToOne` / `@OneToMany` | `@DocumentReference` ou documento embutido — [aqui, embutido](./desnormalizacao-mongo.md) |
| Schema criado por migration (Flyway) | Sem schema — a coleção nasce no primeiro insert |

**Atenção ao import do `@Id`.** É `org.springframework.data.annotation.Id`, não `jakarta.persistence.Id`. Como o projeto costuma ter as duas dependências no classpath em outros serviços, a IDE sugere a errada com facilidade — e o sintoma é o Mongo gerar um `ObjectId` próprio e ignorar o seu campo.

O `@EqualsAndHashCode(onlyExplicitlyIncluded = true)` com `@EqualsAndHashCode.Include` só no `id` mantém a regra de DDD: **duas entidades são iguais quando têm a mesma identidade**, não quando têm os mesmos atributos.

> ⚠️ **`onlyExplicitlyIncluded` sem nenhum `@Include` é pior que não anotar nada.** Lombok gera um `equals` que compara zero campos: **todo objeto passa a ser igual a todo outro**, e um `Set` deles guarda exatamente um. Foi o que aconteceu com o `StockMovement` recém-criado, e não custou nada porque nada o colocava em coleção — até custar. O par de anotações só funciona junto.

### Uma segunda coleção: `stock_movements`

A Fase 14 acrescentou `StockMovement`, o extrato de entradas e saídas de estoque. Ele vive no mesmo pacote do `Product` e é deliberadamente o oposto dele:

| | `Product` | `StockMovement` |
|---|---|---|
| `AbstractAggregateRoot` | sim — registra eventos | **não** |
| `@Version` | sim — lock otimista | **não** |
| Auditoria (`@CreatedDate`…) | sim | **não** — `occurredAt` é o dado, não metadado |
| Alterado depois de criado | sim | **nunca** |

Nada disso é economia de esforço. **Um registro imutável de fato consumado não tem invariante para proteger** — toda a maquinaria do agregado existe para defender regra sobre estado que muda, e aqui não há estado que mude. Copiar o desenho do `Product` por simetria seria cerimônia sem função. Ver [`transacoes-mongo.md`](./transacoes-mongo.md).

---

## 2. Relacionamento por referência — `@DocumentReference`

> 🔧 **Esta decisão foi REVERTIDA na Fase 12.** A escolha registrada abaixo — referenciar — deixou de valer: hoje a categoria é **embutida** (a Opção A), e o `@DocumentReference` está comentado no código. A seção continua aqui porque o raciocínio dela é o mais instrutivo do documento: a regra prática do fim estava certa, e mesmo assim a conclusão saiu errada. A comparação completa dos dois jeitos, e o que a inversão custou, estão em [`desnormalizacao-mongo.md`](./desnormalizacao-mongo.md).

A decisão de modelagem mais importante do documento. Em Mongo existem dois caminhos para representar "produto pertence a uma categoria":

**Opção A — embutir (embedded) — *a escolhida a partir da Fase 12*:**
```json
{ "_id": "...", "name": "Notebook", "category": { "_id": "0197f2c1-...", "name": "Eletrônicos", "enabled": true } }
```
Leitura em uma única ida ao banco. Mas renomear uma categoria exige atualizar **todos** os produtos dela — o que hoje é feito por evento, de forma assíncrona (ver [`eventos-e-listeners.md`](../01-arquitetura-design/eventos-e-listeners.md)).

**Opção B — referenciar — *a escolhida originalmente, revertida depois*:**
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

**O custo:** cada leitura de produto que toca `getCategory()` dispara uma consulta extra na coleção `categories`. Em uma listagem de 20 produtos isso vira 20 consultas — o clássico **N+1**.

> ✅ **Resolvido em duas etapas.** Primeiro a listagem deixou de usar `find()` e passou a montar um aggregation pipeline com `$lookup`, trazendo a categoria de todos os produtos da página numa ida só — mas o N+1 continuava para quem carregasse `Product` pelo repositório e tocasse `getCategory()`.
>
> A Fase 12 acabou com ele de vez: **não há mais referência a resolver**, então nenhum caminho de leitura pode disparar consulta extra. Ver [`desnormalizacao-mongo.md`](./desnormalizacao-mongo.md).

> Regra prática: **embuta o que é lido junto e muda junto; referencie o que tem vida própria.**

> #### Nota de estudo: a regra estava certa, a leitura dela é que estava apressada
>
> Repare que a Fase 12 **não** desmentiu a regra prática acima — ela apenas a aplicou melhor. O erro original foi deixar *"tem vida própria"* ganhar de *"é lido junto"*, quando os dois critérios respondem perguntas diferentes: um decide se a categoria é um **agregado**, o outro decide se uma **cópia** dela pode viver em outro documento. `Category` continua sendo um agregado; o que está embutido no produto não é a categoria, é uma fotografia dela.
>
> Vale guardar as duas versões, porque a inversão resume um erro que se repete: aplicar a regra certa na ordem errada, e só descobrir quando a conta de leitura chega.

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

> 🔧 **Não é a única estratégia — e na Fase 13 o estoque escolheu outra.** Lock otimista **detecta** o conflito e faz alguém repetir a operação; para saque de estoque não há decisão a repetir, então o ajuste passou a usar **atualização condicional atômica** (`findAndModify` com a regra dentro do filtro), sem carregar o agregado. O `@Version` descrito acima continua valendo para `update`, `enable` e `disable`, que passam pelo repositório. A comparação das três estratégias está em [`concorrencia-e-atomicidade.md`](./concorrencia-e-atomicidade.md).

---

## 7. Regra de negócio dentro do agregado

O `Product` **não** é um saco de getters e setters. Ele protege os próprios invariantes.

Na primeira versão, cada setter público guardava a regra sozinho:

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

Na Fase 12 os setters de preço ficaram **privados** e a operação de negócio virou `changePrice(regularPrice, salePrice)`, que recebe os dois de uma vez. A regra saiu dos setters e passou a viver em um método só:

```java
// domain/product/Product.java
private void validatePrices(BigDecimal regularPrice, BigDecimal salePrice) {
    if (salePrice.compareTo(regularPrice) > 0) {
        throw new DomainException("Sale price cannot be greater than regular price");
    }
}
```

chamado pelo construtor e pelo `changePrice`, sempre sobre o **par completo**.

> #### Nota de estudo: mover um invariante é onde ele se perde
>
> A mudança acima nasceu quebrada, e vale muito mais documentada do que escondida. Ao passar a regra dos setters para o `changePrice`, ela ficou escrita assim:
>
> ```java
> if (regularPrice.compareTo(this.salePrice) < 0) {   // this.salePrice é o ANTIGO
> ```
>
> O parâmetro novo sendo comparado com o campo **antigo**. Isso errava nas duas direções:
>
> | Chamada, partindo de `(3000, 2789)` | O que acontecia | O que deveria |
> |---|---|---|
> | `changePrice(2500, 2400)` | rejeitava — `2500 < 2789` | aceitar: `2400 ≤ 2500` |
> | `changePrice(3000, 5000)` | **aceitava** — `3000 < 2789` é falso | rejeitar: `5000 > 3000` |
>
> O segundo caso gravava promoção mais cara que o preço cheio, com `discountPercentageRounded` em **-67%**. E o caminho de criação ficou sem validação nenhuma, porque as guardas tinham saído dos setters e só o `changePrice` recebeu uma.
>
> A lição não é "revise melhor": é que **um setter enxerga metade da regra**. Enquanto o invariante depende de dois campos, guardá-lo em qualquer método que receba só um é acidente esperando acontecer — e o teste que pega isso é o de agregado puro, sem Spring (`src/test/.../domain/product/ProductTest.java`).

Três lições concretas:

1. **`compareTo`, nunca `equals`, com `BigDecimal`.** `new BigDecimal("10.0").equals(new BigDecimal("10.00"))` é `false` — o `equals` compara também a escala. `compareTo` compara valor.
2. **O invariante mora no agregado — e num ponto só.** "Preço promocional não pode ser maior que o preço normal" é verificado em `validatePrices`. Não existe caminho — nem application service, nem controller — que consiga criar ou alterar um produto violando isso.
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
| Descrição | `shortDescription` (50 caracteres, via `$substrCP` no banco) | `description` completa |
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

### O `TypeMap` do `ProductSummaryOutput` foi removido depois

O bloco acima registra o estado desta etapa. Quando a listagem virou aggregation, o `$project` passou a devolver os documentos **já no formato do DTO** — não havia mais nada para o mapper converter, e o `TypeMap` do `ProductSummaryOutput` saiu.

Restou só o do `ProductDetailOutput`, e a diferença é instrutiva: os dois DTOs são preenchidos por caminhos opostos, de propósito.

| DTO | Caminho | Quem deriva os campos |
|---|---|---|
| `ProductDetailOutput` | `findById` → `find()` → mapper | Java, com `Converter` |
| `ProductSummaryOutput` | `filter` → `aggregate()` | MongoDB, no `$project` |

O único derivado que **não** migrou foi o `slug`, que continua em Java — agora num getter do próprio DTO, porque tirar acento com operador do Mongo custaria uma cadeia de `$replaceAll`. Ver [`agregacoes-mongo.md`](./agregacoes-mongo.md#project-derivar-no-servidor).

E o slug em si (o `Slugfier` mudou de `infrastructure/util` para `application/util`, já que é função pura de `String` e passou a ser usado por um DTO de `application`):

```java
// application/util/Slugfier.java
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
# src/main/resources/application-development-env.yml
spring:
  mongodb:
    uri: mongodb://localhost:27017,localhost:27018,localhost:27019/product_catalog?replicaSet=rs0
```

> 🔧 **Duas mudanças na Fase 14.** A configuração foi quebrada em três arquivos — `application.yml` (só os grupos de perfil), `application-base.yml` (o que vale em qualquer ambiente) e `application-development-env.yml` (só a URI e a carga). E a URI virou multi-seed com `replicaSet=rs0`, sem credenciais: o cluster local passou a rodar sem autenticação. O porquê dos três nós está em [`transacoes-mongo.md`](./transacoes-mongo.md).

> ⚠️ **Armadilha de versão.** O projeto usa **Spring Boot 4.0**, onde a autoconfiguração do MongoDB saiu de `spring-boot-autoconfigure` para o módulo `spring-boot-mongodb` (pacote `org.springframework.boot.mongodb.autoconfigure`) e a propriedade passou de `spring.data.mongodb.uri` para **`spring.mongodb.uri`**.
>
> Praticamente todo tutorial e resposta de fórum ainda usa `spring.data.mongodb.*`. Se copiar de lá, a propriedade é silenciosamente ignorada e a aplicação conecta no default (`localhost:27017/test`, sem autenticação) — o sintoma é "conecta, mas a coleção está sempre vazia".

Enquanto o Mongo teve autenticação, a URI precisava de `?authSource=admin`: o usuário `root` era criado no banco `admin` (`MONGO_INITDB_ROOT_USERNAME` no compose), mas os dados vivem em `product_catalog` — sem isso o Mongo procura as credenciais no banco errado e recusa a conexão. Vale conhecer o sintoma, porque ele reaparece em qualquer ambiente com auth ligada.

Para subir o Mongo local: [`../04-infraestrutura/ambiente-local.md`](../04-infraestrutura/ambiente-local.md).

---

## 11. Índices declarados no agregado

Numa etapa posterior o `Product` ganhou índices, e eles são declarados **na própria classe** — a definição do índice mora ao lado da modelagem, não num script separado:

```java
@Document(collection = "products")
@CompoundIndex(name = "pidx_product_by_category_enabledTrue_addedAt",
        def = "{'category.id': 1, 'enabled': 1, 'addedAt': -1}",
        partialFilter = "{'enabled': true}")
public class Product extends AbstractAggregateRoot<Product> {

    @TextIndexed(weight = 1)
    private String name;

    @Indexed(name = "idx_product_by_brand")
    private String brand;
    ...
}
```

É uma diferença cultural em relação ao lado relacional do projeto: no `ordering` e no `billing` o índice nasce numa migration do [Flyway](./flyway.md), versionada e explícita. Aqui ele nasce de anotação, criada na subida por `spring.data.mongodb.auto-index-creation: true` — conveniente para estudar, inadequado para produção pelo mesmo motivo que `ddl-auto` é.

**Mecânica, ESR, índice parcial, índice de texto e como conferir com `explain`:** [`indices-mongo.md`](./indices-mongo.md).

### `@TextScore` — um campo que não é do documento

```java
@TextScore
private Float score;
```

O `score` não existe na coleção. É preenchido pelo Mongo a cada busca textual com a relevância daquele documento naquela consulta, e chega `null` em qualquer outra leitura.

Vale a pergunta honesta: **isso deveria estar no agregado de domínio?** É um dado de infraestrutura de consulta — a relevância de um resultado de busca não é atributo do produto. O mesmo raciocínio que manteve HTTP fora do domínio serviria para deixar o `score` só na projeção de leitura. Está no agregado porque é o que o Spring Data exige para popular o campo; é uma concessão consciente ao framework, do mesmo tipo que o `@Document` já era.

### Construtor sem argumentos protegido

```java
@NoArgsConstructor(access = AccessLevel.PROTECTED)
```

O Spring Data precisa de um construtor sem argumentos para materializar o documento, mas ele instancia por reflexão e não se importa com a visibilidade. Deixá-lo `protected` fecha a porta para código de aplicação criar um `Product` vazio e preenchê-lo por fora, desviando do builder e dos invariantes. Mesmo padrão já usado no `Category`.

---

## Pendências registradas

Coisas que ficaram conscientemente pela metade nesta etapa:

- ✅ ~~`ProductQueryServiceImpl.filter()` ainda retorna `null`~~ — **implementado** com `Query` + `Criteria` e paginação manual; ver [`consultas-mongo-criteria.md`](./consultas-mongo-criteria.md).
- [ ] Não há testes cobrindo o agregado `Product` (nem os invariantes de preço, nem a persistência). O `ProductRepositoryIT` que surgiu depois exercita só a projeção do repositório, e sem asserção — apenas loga o resultado.
- [ ] `auditorProvider()` devolve `UUID.randomUUID()` até existir autenticação.
- [x] ~~`quantityInStock` tem `setQuantityInStock` **privado** e nenhum método público de entrada/saída de estoque. Produto criado **pela API** fica travado em `0`, e `isInStock()` sempre retorna `false`.~~ Resolvido na Fase 13 com `POST /{productId}/restock` e `/withdraw` — e resolvido **sem** setter público: o ajuste acontece direto no banco, de forma atômica, sem carregar o agregado. Ver [`concorrencia-e-atomicidade.md`](./concorrencia-e-atomicidade.md). O detalhe histórico continua valendo: os documentos do [`DataLoader`](../04-infraestrutura/carga-de-dados-mongo.md) são inseridos crus, e era por isso que `?inStock=true` funcionava na massa de teste e não num produto cadastrado pelo endpoint.
- [x] ~~N+1 latente no `@DocumentReference`.~~ Resolvido de vez: a listagem primeiro trocou o N+1 pelo `$lookup` (ver [`agregacoes-mongo.md`](./agregacoes-mongo.md)), e a Fase 12 removeu a referência inteira — não há mais o que resolver em nenhum caminho de leitura. Ver [`desnormalizacao-mongo.md`](./desnormalizacao-mongo.md).
- [ ] `brand` tem `@Indexed` e nenhuma consulta filtra por marca — índice que só custa escrita. Detalhe em [`indices-mongo.md`](./indices-mongo.md).

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
- [ ] …mas `auto-index-creation` continua em `spring.data.mongodb` — os dois prefixos convivem

---

## Referências

- [Spring Data MongoDB — Reference](https://docs.spring.io/spring-data/mongodb/reference/)
- [MongoDB — Data Model Design (embed vs. reference)](https://www.mongodb.com/docs/manual/core/data-model-design/)
- [RFC 9562 — UUID v7](https://www.rfc-editor.org/rfc/rfc9562)
- [`consultas-mongo-criteria.md`](./consultas-mongo-criteria.md) — como consultar esse modelo
- [`agregacoes-mongo.md`](./agregacoes-mongo.md) — o pipeline que resolveu o N+1 do `@DocumentReference`
- [`desnormalizacao-mongo.md`](./desnormalizacao-mongo.md) — a reversão da decisão da seção 2, e o que ela custou
- [`concorrencia-e-atomicidade.md`](./concorrencia-e-atomicidade.md) — a outra estratégia de concorrência, e a escrita que não passa pelo repositório
- [`eventos-e-listeners.md`](../01-arquitetura-design/eventos-e-listeners.md) — os eventos de domínio do `Product` e a propagação da cópia da categoria
- [`indices-mongo.md`](./indices-mongo.md) — os índices declarados neste agregado
- [`carga-de-dados-mongo.md`](../04-infraestrutura/carga-de-dados-mongo.md) — como popular a coleção
