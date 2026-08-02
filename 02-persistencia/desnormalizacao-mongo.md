# Normalizado × Desnormalizado no MongoDB

> Como a categoria deixou de ser uma referência (`categoryId`) e virou uma **cópia embutida** dentro de cada produto — o que isso resolveu de uma vez, e a conta que passou a chegar todo mês.
> Código real: `domain/product/ProductCategory.java`, `domain/product/Product.java`, `infrastructure/persistence/category/ProductCategoryUpdater.java`.

> Este documento trata da **modelagem**. A consequência dela — manter a cópia em dia — está em [`eventos-e-listeners.md`](../01-arquitetura-design/eventos-e-listeners.md). Os dois se leem em sequência: desnormalizar cria o problema que o outro doc resolve.

---

## O problema

A [Fase 8](../00-visao-geral/linha-do-tempo.md) escolheu **referenciar** a categoria:

```java
@DocumentReference
@Field(name = "categoryId")
private Category category;
```

O documento gravado ficava assim:

```json
{
  "_id": { "$uuid": "946cea3b-..." },
  "name": "HyperNova Pro X11",
  "categoryId": { "$uuid": "e0c4271d-..." }
}
```

Limpo, sem duplicação — e caro para ler. Toda listagem precisa do **nome** da categoria, e o nome não estava ali. Cada produto que tocasse `getCategory().getName()` disparava uma consulta: numa página de 15, dezesseis idas ao banco.

A [Fase 11](../00-visao-geral/linha-do-tempo.md) atacou o sintoma com um `$lookup` no aggregation pipeline. Funcionou: dezesseis idas viraram uma. Mas o join continuava lá, rodando a cada listagem, sobre todos os documentos filtrados — e trouxe junto a pegadinha do `$unwind` sem flag, que é *inner join* e faz produto de categoria órfã desaparecer sem aviso.

A Fase 12 desiste de otimizar a leitura e ataca a **modelagem**: se o nome da categoria é lido em toda listagem, ele devia estar no documento.

---

## A cópia embutida

`ProductCategory` é um value object — não tem `@Document`, não tem coleção própria, não tem ciclo de vida próprio:

```java
// domain/product/ProductCategory.java
@Getter
@AllArgsConstructor
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class ProductCategory {
    private UUID id;
    private String name;
    private Boolean enabled;

    public static ProductCategory of(Category category) {
        return new ProductCategory(category.getId(), category.getName(), category.getEnabled());
    }
}
```

O `Product` guarda uma instância dele, e o `setCategory` recebe a `Category` de verdade só para extrair a cópia:

```java
// domain/product/Product.java
private ProductCategory category;

public void setCategory(Category category) {
    Objects.requireNonNull(category);
    this.category = ProductCategory.of(category);
}
```

O documento passa a ser:

```json
{
  "_id": { "$uuid": "946cea3b-..." },
  "name": "HyperNova Pro X11",
  "category": {
    "_id": { "$uuid": "e0c4271d-..." },
    "name": "Laptops",
    "enabled": true
  }
}
```

E o pipeline perde dois estágios inteiros:

```java
// infrastructure/persistence/product/ProductQueryServiceImpl.java
operations.addAll(Arrays.asList(
//                lookup("categories", "categoryId", "_id", "category"),
//                unwind("$category"),
        sort(sortWith(filter)),
        projectionForSummary(),
        skip(pageRequest.getOffset()),
        limit(filter.getSize())
));
```

Ficaram comentados de propósito, como referência de estudo — é a versão normalizada, e vale poder olhar as duas lado a lado.

> **Copie o mínimo.** `ProductCategory` tem três campos porque são os três que a listagem lê. Cada campo a mais é um campo a mais para sincronizar a cada rename, e um documento de produto maior em disco e em memória. A pergunta para incluir um campo novo não é *"o produto tem acesso a esse dado?"* — é *"vale reescrever N produtos quando ele mudar?"*.

---

## A comparação

| | Normalizado (`@DocumentReference`) | Desnormalizado (embutido) |
|---|---|---|
| Ler 15 produtos com o nome da categoria | 16 idas ao banco, ou 1 com `$lookup` | **1 ida, sem join** |
| Renomear uma categoria | 1 escrita | 1 escrita **+ N produtos** |
| Consistência da cópia | não existe cópia — sempre exata | **eventual** |
| Filtrar por categoria | `categoryId` | `category.id` → `category._id` |
| `count` × pipeline | podiam divergir (`$unwind` = *inner join*) | não têm mais como discordar |
| Categoria órfã (id que não resolve) | produto **sumia** da listagem, silenciosamente | produto aparece, com a cópia que tinha |
| Tamanho do documento | menor | maior — o nome da categoria repetido em cada produto |
| Custo de escrita nos índices | rename não toca índice de produto | rename escreve em campo indexado de N produtos |

A leitura ficou mais barata e mais simples. A escrita ficou mais cara e passou a ter uma janela de inconsistência. **Não é um upgrade — é uma troca**, e ela só compensa porque o catálogo é lido muito mais do que escrito.

---

## A regra não mudou; a aplicação dela é que estava errada

O [`product-catalog-mongo.md`](./product-catalog-mongo.md) já registrava a regra desde a Fase 8:

> Regra prática: **embuta o que é lido junto e muda junto; referencie o que tem vida própria.**

A regra estava certa. A leitura dela é que tinha sido apressada — "categoria tem vida própria" ganhou de "categoria é lida junto", quando os dois critérios da regra dizem respeito a coisas diferentes:

| Pergunta | Categoria do produto |
|---|---|
| É lido junto? | **Sempre.** Não existe listagem de produto sem o nome da categoria |
| Muda junto? | Não muda **quase nunca** — um rename é evento raro |
| Tem vida própria? | Sim, mas isso decide se ela é um **agregado**, não se uma cópia dela pode viver em outro lugar |

O erro foi tratar "tem vida própria" como impeditivo de cópia. `Category` continua sendo um agregado, com coleção, id e ciclo de vida próprios — o que está embutido no produto **não é a categoria**, é uma fotografia dela.

> #### Nota de estudo: o instinto relacional atrapalha aqui
>
> Quem vem de SQL lê "duplicar dado" como defeito, porque em banco relacional duplicação **é** defeito: o modelo não tem como manter as cópias em dia, então a normalização é a única defesa. Num banco de documentos, a pergunta não é *"há duplicação?"* e sim *"quem é o dono do dado, e quem paga por mantê-lo em dia?"*. A resposta aqui é explícita: o dono é a coleção `categories`, e quem paga é o `CategoryEventListener`. Enquanto houver um dono claro e um mecanismo declarado, a cópia é uma decisão — não uma bagunça.
>
> A base conceitual disso está em [`nosql-conceitos.md`](./nosql-conceitos.md), que já dizia: *"relacionais favorecem normalização (3FN); NoSQL geralmente favorece desnormalização (duplicar dados para evitar JOINs)"*. Este documento é essa frase virando código.

---

## ⚠️ `category.id` em Java, `category._id` no banco

Aqui mora a pegadinha que mais confunde nesta etapa: **o nome do campo não é o mesmo dos dois lados**.

Toda propriedade chamada `id` — inclusive dentro de um objeto **embutido** — vira `_id` no documento. É o Spring Data aplicando a mesma convenção do `@Id` do agregado raiz. Então o índice declarado assim:

```java
// domain/product/Product.java
@CompoundIndex(name = "pidx_product_by_category_enabledTrue_salePrice",
        def = "{'category.id': 1, 'enabled': 1, 'salePrice': 1}",
        partialFilter = "{'enabled': true}")
```

nasce, de fato, assim:

```
$ docker exec algashop-meta-algashop-mongodb-1 mongosh ... \
    --eval 'db.products.getIndexes()'

{"category._id":1,"enabled":1,"salePrice":1}   name=pidx_product_by_category_enabledTrue_salePrice
{"category._id":1,"enabled":1,"addedAt":-1}    name=pidx_product_by_category_enabledTrue_addedAt
```

Quem traduz é o **mapping context**: ele resolve o caminho da propriedade Java para o nome do campo no documento. E a tradução vale em mais lugares do que se imagina:

| Onde | O que se escreve | O que chega no Mongo | Quem traduz |
|---|---|---|---|
| `@CompoundIndex(def = ...)` | `category.id` | `category._id` | resolvedor de índices |
| `Criteria` do `$match` | `category.id` | `category._id` | `QueryMapper` |
| `Criteria` do `updateMulti` | `category._id` (cru) | `category._id` | — já é o nome final |
| `$project` | `category._id` (cru) | `category._id` | — já é o nome final |

Ou seja: **as duas grafias funcionam**, e é por isso que o código convive com as duas sem quebrar. Escrever `category.id` é falar a língua do Java e deixar o framework traduzir; escrever `category._id` é falar direto com o banco. Nenhuma das duas está errada — o que não pode é *acreditar que o índice está em `category.id`* e sair procurando por ele no `getIndexes()`.

> **Confira, não deduza.** `db.products.getIndexes()` é a única fonte de verdade sobre o que existe de fato. Índice declarado com nome de campo que o documento não tem não dá erro: ele simplesmente é criado sobre um caminho inexistente e nunca é escolhido pelo planejador.

---

## O que sobrou do aggregation pipeline

Sem `$lookup` e sem `$unwind`, o pipeline atual é:

```
$match → ($addFields score) → $sort → $project → $skip → $limit
```

Ele **continua justificado** — só que por outro motivo. O que o segura de pé agora é o `$project`, que calcula no servidor o que antes era conversor do ModelMapper rodando em Java:

```java
.andExpression("salePrice < regularPrice").as("hasDiscount")
.andExpression("quantityInStock > 0").as("inStock")
.and(StringOperators.valueOf("description").substringCP(0, 50)).as("shortDescription")
```

Duas consequências que valem registrar:

1. **A divergência entre `count` e resultado acabou.** Ela existia porque o `$unwind` sem `preserveNullAndEmptyArrays` é *inner join*: um produto de categoria órfã entrava na contagem (feita com `Query` comum, sem pipeline) e sumia do resultado. Sem join, as duas contas veem exatamente o mesmo conjunto.
2. **O comportamento com categoria órfã inverteu.** Antes o produto sumia; agora ele aparece, exibindo a cópia que tem. Qual dos dois é "melhor" depende do que se prefere: sumir silenciosamente ou mostrar dado possivelmente velho. O segundo é mais honesto — o produto existe, e a cópia diz *quando* ela era verdade.

Detalhe do `$project` que acompanhou a mudança: `category.enabled` passou a ser projetado, e `CategoryMinimalOutput` ganhou o campo. O contrato em [`../openapi/product-catalog.yml`](../openapi/product-catalog.yml) **já declarava `enabled` como `required`** havia tempo — era a implementação que estava atrás do contrato, não o contrário.

---

## O `_class` que apontava para o nada

A massa de teste gravava, em cada produto:

```json
"_class": "com.algaworks.algashop.product.catalog.domain.model.product.Product"
```

Esse pacote — com o `.model.` no meio — **não existe** neste projeto. O real é `...domain.product.Product`.

E nada quebrou. O `_class` é a dica de tipo que o Spring Data grava para saber em que classe materializar o documento; quando o valor não resolve, o `SimpleTypeInformationMapper` engole o `ClassNotFoundException` e cai no tipo que a consulta pediu. Como toda leitura aqui já pede `Product.class`, o fallback dá no mesmo.

Por que corrigir, então:

- é dado errado versionado no repositório, e o próximo a copiar o arquivo propaga o erro;
- o silêncio é o problema. No dia em que o agregado tiver herança — `Product` e uma subclasse `DigitalProduct`, por exemplo — o `_class` deixa de ser decorativo e passa a decidir a classe materializada. Aí um valor que não resolve vira o tipo errado, sem aviso.

---

## Armadilhas

1. **Cópia embutida não é atualizada por quem escreve o dono.** Salvar uma `Category` não toca em produto nenhum. É preciso alguém propagar — e é isso que o [`eventos-e-listeners.md`](../01-arquitetura-design/eventos-e-listeners.md) documenta.
2. **`category.id` no Java, `category._id` no documento.** Verifique com `getIndexes()` antes de concluir que um índice não está sendo usado.
3. **A carga de dados escreve `category` cru.** O `DataLoader` insere o JSON como está, sem passar por `ProductCategory.of(...)` — nada garante que o `name` gravado no produto bata com o da coleção `categories`. A massa pode nascer já dessincronizada.
4. **O `updateMulti` da propagação não usa índice.** Ele filtra só por `category._id`, sem `enabled: true`, então não casa o `partialFilter` dos dois índices compostos — é varredura.
5. **Escrever a cópia mexe em campo indexado.** `category.name` e `category.enabled` estão nos índices compostos; um rename reescreve documento **e** índice, N vezes.
6. **A cópia não sobe de versão.** O `updateMulti` escreve por `MongoOperations`, direto: o `@Version` do produto não é incrementado e a alteração escapa do lock otimista.

---

## Pendências registradas

- [ ] **A propagação é uma varredura.** `ProductCategoryUpdater` filtra por `category._id` sem `enabled`, então nenhum dos dois índices parciais serve. Um índice simples em `category._id`, sem `partialFilter`, resolveria — ao custo de mais um índice para manter.
- [ ] **A massa de teste pode nascer dessincronizada.** `products.json` traz `category.name` escrito à mão; nada valida contra `categories.json`. Ver [`carga-de-dados-mongo.md`](../04-infraestrutura/carga-de-dados-mongo.md).
- [ ] **Não há reconciliação.** Se a propagação falhar, nada percebe e nada corrige depois. Uma rotina de varredura comparando `category.name` com a coleção `categories` seria a rede de segurança — hoje não existe.
- [ ] **`@Version` não acompanha a propagação.** Escrita por `MongoOperations` não incrementa a versão; duas alterações concorrentes no mesmo produto não são detectadas.
- [ ] **O `$project` ainda roda antes do `$skip`/`$limit`.** Herdado da Fase 11 e não resolvido aqui — projetar depois de paginar seria mais barato. Ver [`agregacoes-mongo.md`](./agregacoes-mongo.md).
- [x] ~~**N+1 no `@DocumentReference`.**~~ Resolvido de vez: não há mais referência a resolver. O `$lookup` da Fase 11 tinha resolvido só para a listagem; agora vale para qualquer caminho.
- [x] ~~**`count` e pipeline podiam divergir.**~~ Resolvido — a divergência vinha do `$unwind`, que não existe mais.
- [x] ~~**`_class` da massa apontando para pacote inexistente.**~~ Corrigido nesta etapa.

---

## Checklist de revisão

- [ ] Sei explicar a diferença entre embutir e referenciar, e o que cada um cobra
- [ ] Sei por que o `$lookup` foi aposentado sem ter dado errado
- [ ] Entendo que `ProductCategory` é uma cópia, e que a `Category` continua sendo o dono
- [ ] Sei por que só três campos foram copiados, e qual é a pergunta antes de copiar o quarto
- [ ] Sei que `category.id` vira `category._id`, e sei conferir isso no `getIndexes()`
- [ ] Entendo por que a contagem e o resultado não podem mais divergir
- [ ] Sei o que acontece hoje com um produto cuja categoria foi apagada
- [ ] Sei explicar o que é o `_class` e por que um valor errado passou despercebido
- [ ] Entendo que a leitura ficou barata às custas da escrita, e por que essa troca compensa num catálogo

---

## Referências

- [MongoDB — Data Model Design: embedding vs. referencing](https://www.mongodb.com/docs/manual/core/data-model-design/)
- [MongoDB — Model One-to-Many Relationships with Embedded Documents](https://www.mongodb.com/docs/manual/tutorial/model-embedded-one-to-many-relationships-between-documents/)
- [Spring Data MongoDB — Mapping](https://docs.spring.io/spring-data/mongodb/reference/mongodb/mapping/mapping.html)
- [`eventos-e-listeners.md`](../01-arquitetura-design/eventos-e-listeners.md) — quem mantém a cópia em dia, e a consistência eventual que isso traz
- [`product-catalog-mongo.md`](./product-catalog-mongo.md) — a modelagem do agregado e a decisão original de referenciar
- [`agregacoes-mongo.md`](./agregacoes-mongo.md) — o pipeline, e o `$lookup` que esta etapa aposentou
- [`indices-mongo.md`](./indices-mongo.md) — os índices compostos que passaram a começar em `category._id`
- [`nosql-conceitos.md`](./nosql-conceitos.md) — normalização × desnormalização e o BASE, em teoria
