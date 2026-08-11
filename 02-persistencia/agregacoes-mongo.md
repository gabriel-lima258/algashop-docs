# Aggregation Pipeline no MongoDB

> Como a listagem de produtos deixou de ser `find()` + ModelMapper e virou um pipeline de estágios que junta coleção, calcula campo derivado e projeta — tudo numa ida só ao banco.
> Código real: `infrastructure/persistence/product/ProductQueryServiceImpl.java`.

> Este é o **jeito 2** de consultar. O jeito 1, com `Query` + `Criteria`, continua vivo e documentado em [`consultas-mongo-criteria.md`](./consultas-mongo-criteria.md) — a seção [Quando usar cada um](#quando-usar-cada-um) compara os dois.

> 🔧 **Mudou na Fase 12.** O `$lookup` e o `$unwind` descritos aqui foram **aposentados**: a categoria deixou de ser referência e passou a ser uma cópia embutida no próprio documento, então não há mais nada a juntar. O pipeline continua de pé, agora justificado só pelo `$project`. As seções sobre o join permanecem neste documento de propósito — mostram o que existiu, e por quê. Ver [`desnormalizacao-mongo.md`](./desnormalizacao-mongo.md).
>
> O pipeline em produção hoje é: `$match → ($addFields score) → $sort → $project → $skip → $limit`.

---

## O problema

A listagem precisa mostrar, por produto: o **nome da categoria**, se tem desconto, se tem estoque e um resumo da descrição.

Com `find()` + ModelMapper, cada um desses custa caro por um motivo diferente:

| O que a tela precisa | Como saía antes | Custo |
|---|---|---|
| Nome da categoria | `getCategory().getName()` via `@DocumentReference` | **uma consulta extra por produto** — o clássico N+1 (resolvido de vez na Fase 12, ver callout acima) |
| `hasDiscount`, `inStock` | calculado em Java, depois da consulta | o documento inteiro trafega, mesmo o que não se usa |
| `shortDescription` | conversor do ModelMapper | idem — a descrição completa vem pela rede para ser cortada em Java |

Numa página de 15 produtos, isso é **1 + 15 = 16 idas ao banco**, trazendo campos que serão descartados.

O aggregation pipeline resolve os três de uma vez: junta a categoria, calcula os derivados e recorta o documento — no servidor, numa consulta.

---

## O que é um pipeline

Uma **esteira de estágios**. Cada estágio recebe a saída do anterior e devolve documentos transformados para o próximo.

```
documentos da coleção
   → $match     (filtra)
   → $addFields (acrescenta campo)
   → $lookup    (junta outra coleção)
   → $unwind    (desembrulha o array do lookup)
   → $sort      (ordena)
   → $project   (escolhe/calcula os campos de saída)
   → $skip/$limit (pagina)
   → resultado
```

Para quem vem de SQL:

| Estágio | Faz | Equivalente SQL |
|---|---|---|
| `$match` | filtra documentos | `WHERE` |
| `$lookup` | junta outra coleção | `LEFT JOIN` |
| `$unwind` | transforma array em linhas | — (não tem equivalente direto) |
| `$addFields` | acrescenta campo calculado, mantendo o resto | coluna calculada |
| `$project` | escolhe e calcula os campos de saída | `SELECT` |
| `$sort` | ordena | `ORDER BY` |
| `$skip` / `$limit` | pagina | `OFFSET` / `LIMIT` |

A diferença central em relação ao SQL: **a ordem é escrita por você e é executada literalmente**. Não há otimizador reescrevendo o plano como um banco relacional faria. Isso é liberdade e é armadilha — ver [A ordem dos estágios é o custo](#-a-ordem-dos-estágios-é-o-custo).

---

## `$match` — o `Criteria` reaproveitado

A boa notícia: **o `Criteria` do jeito 1 continua valendo**. É a mesma API, só que o resultado precisa entrar num estágio.

O que mudou é a montagem:

```java
// jeito 1 — empilha na propria Query
query.addCriteria(Criteria.where("enabled").is(v));
query.addCriteria(Criteria.where("salePrice").gte(x));

// jeito 2 — acumula numa lista e junta num criteria so
List<Criteria> criterias = new ArrayList<>();
criterias.add(Criteria.where("enabled").is(v));
criterias.add(Criteria.where("salePrice").gte(x));
return Optional.of(new Criteria().andOperator(criterias));
```

O `$match` recebe **um** criteria, não uma coleção deles — daí o `andOperator`, que produz um `$and` único.

### Por que `Optional<Criteria>`

```java
Optional<Criteria> criteria = buildCriteria(filter);
criteria.ifPresent(c -> operations.add(Aggregation.match(c)));
```

Se nenhum filtro veio preenchido, a alternativa seria devolver `new Criteria()` — que serializa como `{}` e vira um `$match: {}`: um estágio inútil que percorre tudo para não filtrar nada. O `Optional` deixa "não há filtro" ser uma resposta explícita, e o estágio simplesmente não entra no pipeline.

### O detalhe de tipos que morde: `$expr`

O filtro `hasDiscount` compara dois campos do mesmo documento e precisa de `$expr` (ver [`consultas-mongo-criteria.md`](./consultas-mongo-criteria.md#comparar-dois-campos-do-mesmo-documento)). No jeito 1 isso era:

```java
query.addCriteria(AggregationExpressionCriteria.whereExpr(
        ComparisonOperators.valueOf("$salePrice").lessThan("$regularPrice")));
```

Não funciona aqui. `AggregationExpressionCriteria` implementa `CriteriaDefinition`, **mas não `Criteria`** — e `andOperator` só aceita `Criteria`. A saída é escrever o `$expr` como um criteria comum:

```java
AggregationExpression discountExpression =
        ComparisonOperators.valueOf("$salePrice").lessThan("$regularPrice");

criterias.add(Criteria.where("$expr").is(discountExpression.toDocument()));
```

`$expr` é operador de query de primeira classe, então `where("$expr")` é legítimo — o `toDocument()` só materializa a expressão no BSON que ele espera.

> Detalhe de tipagem que vale guardar: a lista é `List<Criteria>`, não `List<CriteriaDefinition>`. Com a lista mais larga o código compila e estoura só em tempo de execução, num `ArrayStoreException` dentro do `andOperator`.

---

## Busca textual dentro do pipeline

Duas regras que não aparecem em lugar nenhum até a consulta falhar:

**1. `$text` tem que ser o primeiro estágio.** É restrição do MongoDB. Por isso o bloco do texto entra antes do `$match` dos demais filtros:

```java
textCriteria.ifPresent(c -> {
    operations.add(Aggregation.match(c));   // sempre primeiro
    ...
});
criteria.ifPresent(c -> operations.add(Aggregation.match(c)));
```

**2. `@TextScore` não vale dentro do pipeline.** No `find()`, a anotação no campo `score` fazia o Spring Data traduzir `Sort.by("score")` para `{score: {$meta: "textScore"}}`. Num aggregation isso não acontece: o campo precisa ser criado **à mão**.

```java
AggregationOperation addTextScoreField = context ->
        new Document("$addFields", new Document("score", new Document("$meta", "textScore")));
operations.add(addTextScoreField);
```

Repare que é uma **lambda**. `AggregationOperation` é uma interface funcional que devolve o `Document` do estágio — quando o Spring Data não tem uma operação pronta (e não tem para `$addFields` com `$meta`), escreve-se o BSON cru. É a válvula de escape que torna qualquer estágio do Mongo alcançável a partir do Java.

> ⚠️ **A consequência que quase passou batido:** com o `score` virando campo comum, `Sort.by("score")` gera `{$sort: {score: 1}}` — **ascendente**, o menos relevante primeiro. No `find()` isso não acontecia porque o `$meta` do Mongo ordena sempre decrescente. Aqui o `DESC` precisa ser explícito:
>
> ```java
> return Sort.by(Sort.Direction.DESC, "score");
> ```

---

## `$lookup` + `$unwind`: o fim do N+1 *(aposentado na Fase 12)*

> 🔧 **Estes dois estágios não estão mais no pipeline.** Continuam no código, comentados, como referência de estudo. A seção fica porque o raciocínio dela é válido e reaproveitável — só não é mais o que roda aqui. O que os aposentou não foi um defeito, e sim a mudança de modelagem descrita em [`desnormalizacao-mongo.md`](./desnormalizacao-mongo.md): sem categoria em outra coleção, não há o que juntar.

```java
lookup("categories", "categoryId", "_id", "category"),
unwind("$category"),
```

Os quatro argumentos do `lookup`: coleção de destino, campo local, campo remoto, nome do campo de saída.

O `$lookup` sempre devolve um **array** — mesmo casando um único documento. O `$unwind` desembrulha esse array de um elemento em um objeto, que é o formato que o `CategoryMinimalOutput` espera.

O ganho é direto:

| | Idas ao banco, página de 15 |
|---|---|
| `@DocumentReference` + ModelMapper | 1 + 15 |
| `$lookup` | 1 |
| categoria embutida (Fase 12) | 1, e sem join |

> ⚠️ **`unwind` sem `preserveNullAndEmptyArrays` é INNER JOIN.** Produto cujo `categoryId` não resolve — categoria apagada, referência quebrada — **desaparece do resultado**, silenciosamente. Para comportamento de `LEFT JOIN` seria `unwind("$category", true)`. A escolha nunca foi deliberada, e a desnormalização tornou a questão obsoleta: hoje o produto aparece exibindo a cópia que tem.

---

## `$project`: derivar no servidor

```java
project()
        .and("name").as("name")
        // ...
        .and("category._id").as("category._id")
        .and("category.name").as("category.name")
        .and("category.enabled").as("category.enabled")

        .andExpression("salePrice < regularPrice").as("hasDiscount")
        .andExpression("quantityInStock > 0").as("inStock")
        .and(StringOperators.valueOf("description")
                .substringCP(0, 50)).as("shortDescription");
```

A primeira metade repassa campo cru. A segunda **calcula** — e é onde o pipeline substitui o ModelMapper:

| Campo | Antes | Agora |
|---|---|---|
| `hasDiscount` | não existia na listagem | `$project`, no banco |
| `inStock` | não existia na listagem | `$project`, no banco |
| `shortDescription` | `Converter` com `abbreviate(15)` | `$substrCP(0, 50)`, no banco |
| `category` | `@DocumentReference` + ModelMapper (N+1) | repasse puro — vem embutido no documento (era `$lookup` até a Fase 12) |
| `slug` | `Converter` com `Slugfier` | **continua em Java**, num getter do DTO |

O `slug` ficou de fora de propósito: remover acento no Mongo exigiria uma cadeia de `$replaceAll` caractere por caractere, enquanto o `Normalizer.Form.NFD` do Java resolve numa linha. Nem tudo que **pode** ir para o banco **deve**.

> ⚠️ **`substringCP` e não `substring`.** `StringOperators.Substr` gera `$substr`, que é alias de `$substrBytes` e corta por **byte**. Se o corte cair no meio de um caractere UTF-8 — trivial numa descrição em português com `ç`, `ã`, `é` — o MongoDB **devolve erro na consulta**, não texto truncado. `$substrCP` conta *code points*.

### O DTO como destino direto

```java
mongoOperations.aggregate(aggregation, Product.class, ProductSummaryOutput.class)
```

Origem `Product` (de onde ler), destino `ProductSummaryOutput` (em que materializar). Como o `$project` já devolve os campos com os nomes do DTO, **não há ModelMapper no caminho** — foi por isso que o `TypeMap` de `ProductSummaryOutput` saiu do `ModelMapperConfig`.

O `ProductDetailOutput` continua no caminho antigo (`findById` → `find()` → mapper). Os dois convivem de propósito, e a diferença entre eles é exatamente o assunto deste doc.

### Derivar o `$project` do DTO — e a pegadinha

Na Fase 15 os 15 campos listados à mão no `$project` deram lugar a uma linha:

```java
return project(ProductSummaryOutput.class)
        .andExpression("salePrice < regularPrice").as("hasDiscount")
        // ...
```

`project(Class)` monta a lista de campos a partir das propriedades da classe. É bom: um campo novo no DTO passa a sair do pipeline sem ninguém precisar lembrar de acrescentá-lo.

> ⚠️ **A classe tem que ser a MESMA que o pipeline materializa.** A primeira versão dessa mudança escreveu `project(ProductDetailOutput.class)`, enquanto o `aggregate(...)` materializava em `ProductSummaryOutput`. Compila, roda, não erra — e os campos que só existem no destino voltam `null`.
>
> No caso, o campo perdido foi o **`score`**, a relevância do `$text`. A ordenação por relevância continuou funcionando (o `$sort` roda **antes** do `$project`, e ali o campo ainda existe); o que sumiu foi o valor no payload. Nenhum contrato e nenhum teste afirmam nada sobre `score`, então nada acusou.
>
> É o preço da conveniência: derivar do DTO tira a duplicação e, em troca, transforma "qual classe está escrita aqui" numa decisão silenciosa. Os dois DTOs têm nomes parecidos e vivem no mesmo pacote.

---

## ⚠️ A ordem dos estágios é o custo

Esta é a parte que separa "funciona" de "funciona bem". Até a Fase 12 o pipeline estava assim:

```
$match → $lookup → $unwind → $sort → $project → $skip → $limit
```

O `$lookup` e o `$project` rodavam **antes** da paginação. Ou seja: com 5.000 produtos casando o filtro, o Mongo fazia o join e a projeção nos **5.000**, e só então jogava 4.985 fora.

A ordem que faz o mesmo trabalho por muito menos:

```
$match → $sort → $skip → $limit → $lookup → $project
```

Paginar primeiro, juntar depois: o `$lookup` rodaria sobre **15** documentos.

> 🔧 **Metade do problema evaporou na Fase 12.** Sem `$lookup` nem `$unwind`, o pipeline atual é `$match → ($addFields) → $sort → $project → $skip → $limit`. O join caro deixou de existir — mas o `$project` **continua rodando antes do `$skip`/`$limit`**, e recortar campo de 5.000 documentos para devolver 15 continua sendo desperdício. A pendência encolheu; não desapareceu.

E o motivo é o mesmo do doc de índices ([`indices-mongo.md`](./indices-mongo.md)): **só o primeiro `$match` aproveita índice da coleção**. Depois do primeiro estágio, o Mongo está trabalhando sobre um resultado intermediário em memória, onde não há índice nenhum. Um `$sort` colocado depois de um `$lookup` perdia a chance de ser servido pelo índice composto que existe justamente para ele — hoje o `$sort` vem logo após o `$match`, e essa parte melhorou de graça.

Regra prática para montar pipeline:

1. **Filtre o mais cedo possível** — `$match` primeiro, para o índice trabalhar.
2. **Reduza antes de enriquecer** — `$sort`/`$skip`/`$limit` antes de `$lookup`.
3. **Projete por último** — recortar campo de 15 documentos, não de 5.000.

> A reordenação **não foi aplicada** neste código; está registrada como pendência. O motivo de deixá-la visível em vez de corrigi-la em silêncio: o pipeline "certo por acidente" é o que mais aparece em tutorial, e reconhecer o problema vale mais que já encontrar o código pronto.

---

## Paginação e o `count` separado

O total continua vindo de fora do pipeline, com uma `Query` comum:

```java
Query query = new Query();
textCriteria.ifPresent(query::addCriteria);
criteria.ifPresent(query::addCriteria);

long totalElements = mongoOperations.count(query, Product.class);
```

Mesma lógica do jeito 1: contar **antes** de paginar (ver [`consultas-mongo-criteria.md`](./consultas-mongo-criteria.md#paginação-contar-antes-de-paginar)). É mais barato que um `$count` no fim de um pipeline com join e projeção.

> ✅ **Os dois já puderam discordar — não podem mais.** Enquanto havia `$unwind` (inner join), um produto de categoria órfã **entrava na contagem e sumia do resultado**, e a API respondia `totalElements: 100` com 99 itens espalhados nas páginas. Sem join, `count` e pipeline enxergam exatamente o mesmo conjunto. Ver [`desnormalizacao-mongo.md`](./desnormalizacao-mongo.md).

---

## Quando usar cada um

| | `Query` + `Criteria` | Aggregation pipeline |
|---|---|---|
| Complexidade de escrita | baixa | alta |
| Junta outra coleção | não | `$lookup` |
| Campo derivado | em Java, depois da consulta | `$project`, no banco |
| Projeção | `fields` ou ModelMapper | `$project` tipado direto no DTO |
| Uso de índice | direto, na consulta inteira | só no primeiro `$match` |
| Ordem de execução | o Mongo decide | **você** decide, estágio a estágio |
| Onde vive hoje | `findById`, `CategoryQueryServiceImpl` | listagem de produtos |

**Escolha `Query` + `Criteria`** quando a consulta é sobre uma coleção só e os campos derivados são baratos de calcular em Java. É menos código, mais legível, e o índice serve a consulta inteira.

**Escolha aggregation** quando precisa juntar coleções, agrupar/somar, ou quando trazer o documento inteiro para recortar em Java é desperdício mensurável.

O erro comum é escolher aggregation por parecer mais sofisticado. Ele custa mais para escrever, mais para ler, e tira do banco boa parte da liberdade de otimizar.

> #### Nota de estudo: a listagem hoje é um caso de fronteira
>
> Vale olhar o próprio projeto com esse critério. A listagem de produtos escolheu aggregation por causa do `$lookup` — e o `$lookup` foi embora na Fase 12. Do que sobrou, só o `$project` justifica o pipeline: `hasDiscount`, `inStock` e `shortDescription` calculados no servidor, em vez de trazer o documento inteiro para recortar em Java.
>
> É justificativa real, mas mais magra que a original, e conviria reavaliá-la se um dia o `$project` encolher. O ponto de estudo é esse: **a razão que justifica uma ferramenta pode desaparecer sem que ninguém repare**, porque o código continua funcionando. Vale reler a decisão quando a premissa muda, não só quando o código quebra.

---

## O `$project` saiu na Fase 19 — e vale entender o porquê

Trazer a imagem principal para a listagem custou o `$project` inteiro:

```java
// antes: o pipeline materializava o DTO direto
mongoOperations.aggregate(aggregation, Product.class, ProductSummaryOutput.class)

// depois: materializa o agregado e mapeia em Java
List<Product> products = mongoOperations
        .aggregate(aggregation, Product.class, Product.class).getMappedResults();
products.stream().map(p -> mapper.convert(p, ProductSummaryOutput.class)).toList();
```

O motivo é uma limitação real, não preguiça: o `mainImage` do resumo é um **objeto aninhado** cuja `url` se monta concatenando uma propriedade da aplicação (`algashop.mapping.image-storage-url`) ao nome do arquivo. O `$project` não conhece configuração de Spring. Ou a URL passava a ser persistida no documento — trocando um problema por outro pior —, ou a montagem voltava para Java.

O preço:

- **O documento inteiro volta do Mongo** em toda listagem: `description` completa e o `Set<Image>` inteiro, para exibir um resumo e uma miniatura. Numa página de 20, multiplicado por 20.
- **O `shortDescription` mudou de formato.** O `$substrCP(0, 50)` cortava em 50 caracteres crus; o `TypeMap` que voltou usa `abbreviate(..., 15)`. Medido na API depois da mudança: `"15-inch lapt..."` — 12 caracteres e reticências.

É a terceira vez que este pipeline muda de estratégia e um campo muda de comportamento sem que nada acuse: o `$lookup` saiu na Fase 12, o `score` sumiu na Fase 15 por causa de um `project(Class)` apontando para o DTO errado, e agora o resumo encolheu. **Nenhum contrato e nenhum teste afirmam nada sobre esses campos**, e é exatamente por isso que eles conseguem mudar em silêncio.

Ver [armazenamento de arquivos](./armazenamento-de-arquivos.md).

---

## Armadilhas

1. **`$text` precisa ser o primeiro estágio.** Em qualquer outra posição, erro.
2. **`@TextScore` não funciona em pipeline.** Sem `$addFields` com `$meta`, o campo não existe.
3. **`Sort.by("score")` é ascendente.** No pipeline `score` é campo comum; o `DESC` é obrigatório.
4. **`$substr` corta por byte.** Com acento, a consulta **erra**. Use `$substrCP`.
5. **`$unwind` sem flag é inner join.** Documento sem correspondência some sem aviso. *(Não morde mais aqui — não há `$unwind` no pipeline atual.)*
6. **`AggregationExpressionCriteria` não é `Criteria`.** Não entra em `andOperator`.
7. **`new Criteria()` vazio casa tudo.** Por isso o `Optional`.
8. **Só o primeiro `$match` usa índice.** Tudo depois trabalha em memória.
9. **A ordem que você escreve é a ordem que executa.** Não há otimizador reescrevendo o plano.

---

## Pendências registradas

- [ ] **O `$project` ainda roda antes do `$skip`/`$limit`**, recortando campo sobre todo o conjunto filtrado para devolver 15. A correção é reordenar para `$match` → `$sort` → `$skip` → `$limit` → `$project`. *(Encolheu na Fase 12: a metade cara desta pendência, o `$lookup`, deixou de existir.)*
- [x] ~~**`count` e pipeline podem divergir** por causa do `$unwind` como inner join.~~ Resolvido pela desnormalização — sem join, não há divergência possível. Ver [`desnormalizacao-mongo.md`](./desnormalizacao-mongo.md).
- [ ] **`_id` e `score` aparecem duplicados** no `$project`. Inofensivo (o segundo sobrescreve o primeiro), mas é sinal de código montado por tentativa.
- [ ] **O resumo mudou de formato duas vezes, sem decisão explícita nas duas.** Era `abbreviate(15)`; virou `$substrCP(0, 50)` na Fase 11; e voltou para `abbreviate(15)` na Fase 19, quando o `$project` saiu. Nenhum contrato ou teste afirma nada sobre o campo.
- [ ] **`fromStringToShortStringConverter` ficou sem uso** no `ModelMapperConfig` — foi mantido como referência do que o mapper fazia, mas é código morto.
- [~] **O pipeline em si continua sem teste.** O `ProductQueryServiceImplTest` cobre a montagem do `$match` — mocka o `MongoOperations`, faz o `count` devolver zero e inspeciona o `query.getQueryObject()` capturado, o que valida o `$expr` sem precisar de Mongo de pé. Justamente por sair cedo no caminho de zero resultados, ele **não** exercita `$project` nem a ordem dos estágios. Isso pede um teste de integração com `@DataMongoTest`.

---

## Checklist de revisão

- [ ] Sei explicar o que é um estágio e por que a saída de um alimenta o próximo
- [ ] Sei o equivalente SQL de `$match`, `$lookup`, `$project`, `$skip`/`$limit`
- [ ] Sei por que o `$lookup` acabava com o N+1 do `@DocumentReference` — e por que ele foi aposentado sem ter dado errado
- [ ] Sei que `$unwind` sem flag é inner join
- [ ] Sei por que `$text` tem que ser o primeiro estágio
- [ ] Sei por que `@TextScore` não funciona aqui e o que o substitui
- [ ] Sei escrever um estágio que o Spring Data não tem, via lambda de `AggregationOperation`
- [ ] Sei por que `AggregationExpressionCriteria` não entra no `andOperator`
- [ ] Sei em que ordem colocar os estágios, e por quê
- [ ] Sei decidir entre `Query` + `Criteria` e aggregation

---

## Referências

- [MongoDB — Aggregation Pipeline](https://www.mongodb.com/docs/manual/core/aggregation-pipeline/)
- [MongoDB — Pipeline optimization](https://www.mongodb.com/docs/manual/core/aggregation-pipeline-optimization/)
- [MongoDB — `$lookup`](https://www.mongodb.com/docs/manual/reference/operator/aggregation/lookup/)
- [MongoDB — `$substrCP` vs. `$substrBytes`](https://www.mongodb.com/docs/manual/reference/operator/aggregation/substrCP/)
- [Spring Data MongoDB — Aggregation Framework](https://docs.spring.io/spring-data/mongodb/reference/mongodb/aggregation-framework.html)
- [`consultas-mongo-criteria.md`](./consultas-mongo-criteria.md) — o jeito 1, com `Query` + `Criteria`
- [`indices-mongo.md`](./indices-mongo.md) — por que só o primeiro `$match` usa índice
- [`product-catalog-mongo.md`](./product-catalog-mongo.md) — o `@DocumentReference` que gerava o N+1
- [`desnormalizacao-mongo.md`](./desnormalizacao-mongo.md) — a mudança de modelagem que aposentou o `$lookup`
