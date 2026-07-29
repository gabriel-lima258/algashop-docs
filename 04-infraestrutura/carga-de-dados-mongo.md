# Carga de dados no MongoDB

> Como o `product-catalog` popula o banco a cada inicialização — o substituto do Flyway num banco que não tem migration.
> Código real: `infrastructure/persistence/dataload/`.

---

## O problema

No `ordering` e no `billing`, o Flyway resolve duas coisas de uma vez: cria o schema e, se quiser, insere massa inicial. Ver [`flyway.md`](../02-persistencia/flyway.md).

No MongoDB não existe schema para criar — a coleção nasce no primeiro insert. Mas o outro lado do problema continua: **o banco de desenvolvimento nasce vazio**. Sem produto nenhum, não dá para testar listagem, filtro, paginação ou ordenação.

As opções na mesa eram:

| Opção | Por que não |
|---|---|
| Inserir na mão pelo `mongosh` | some no primeiro `docker compose down -v`; ninguém mais no time tem os mesmos dados |
| `mongoimport` num script shell | mais uma ferramenta para instalar, fora do ciclo da aplicação |
| Biblioteca de migration (Mongock) | dependência a mais para um projeto de estudo |
| **Um `ApplicationRunner` que lê JSON do classpath** | ✅ zero dependência nova, versionado junto com o código |

---

## `ApplicationRunner` — o gancho certo

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class DataLoader implements ApplicationRunner {

    private final MongoOperations mongoOperations;
    private final DataLoadProperties properties;

    @Override
    public void run(ApplicationArguments args) {
        if (!properties.getEnabled()) {
            return;
        }
        properties.getSources().forEach(this::importJsonFileToCollection);
    }
}
```

O Spring Boot chama todo `ApplicationRunner` **uma vez, depois que o contexto está completamente pronto** — beans criados, conexões abertas, autoconfiguração aplicada.

Por que não `@PostConstruct`: ele roda durante a inicialização do bean, quando o resto do contexto ainda está subindo. Depender do `MongoOperations` nesse ponto é apostar na ordem de criação dos beans. `ApplicationRunner` não tem essa ambiguidade.

> `CommandLineRunner` serve para o mesmo momento — a diferença é só a assinatura (`String[] args` cru vs. `ApplicationArguments` já parseado).

---

## Configuração externalizada

O que carregar não está no código:

```yaml
# application.yml
algashop:
  data-load:
    enabled: true
    auto-drop: true
    sources:
      - location: db/testdata/categories.json
        collection: categories
      - location: db/testdata/products.json
        collection: products
```

```java
@Component
@ConfigurationProperties("algashop.data-load")
@Data
@Validated
public class DataLoadProperties {

    @NotNull private Boolean enabled;
    @NotNull private Boolean autoDrop;

    @Valid private List<DataLoadSource> sources;

    @Data
    public static class DataLoadSource {
        @NotBlank private String location;
        @NotBlank private String collection;
    }
}
```

Três coisas para reparar:

1. **Kebab-case vira camelCase.** `auto-drop` no YAML, `autoDrop` no Java — o *relaxed binding* do Spring Boot faz a conversão. Também aceita `AUTO_DROP` como variável de ambiente, o que importa em container.
2. **`@Validated` + `@NotNull` derrubam a aplicação na inicialização** se a configuração faltar, com mensagem explícita. É o oposto do valor default silencioso: melhor falhar ao subir do que descobrir um `NullPointerException` no meio da carga.
3. **`@Valid` na lista** propaga a validação para dentro de cada item — sem ele, um `source` com `collection` em branco passaria batido.

A ordem da lista importa: `categories` vem antes de `products` porque o produto referencia a categoria por `categoryId`.

---

## Extended JSON — por que os arquivos não são JSON comum

```json
{
  "_id": { "$uuid": "946cea3b-d11d-4f11-b88d-3089b4e74087" },
  "addedAt": { "$date": "2024-06-14T19:46:33.013Z" },
  "version": { "$numberLong": "2" },
  "regularPrice": { "$numberDecimal": "3000.00" },
  "salePrice": { "$numberDecimal": "2789.00" },
  "quantityInStock": 50,
  "categoryId": { "$uuid": "e0c4271d-0016-4a42-82fe-bf695a9fb9b8" },
  "_class": "com.algaworks.algashop.product.catalog.domain.product.Product"
}
```

JSON tem 6 tipos; BSON tem mais de 20. **Extended JSON** é a notação que o MongoDB usa para representar os tipos extras dentro de um JSON válido.

```java
BsonArray array = BsonArray.parse(rawJson);
return array.stream().map(Object::toString).map(Document::parse).collect(toList());
```

O `BsonArray.parse` entende essa notação. Sem ela:

| Escrito como | Vira | Consequência |
|---|---|---|
| `"946cea3b-..."` | String | `findById(UUID)` não acha nada — tipos diferentes não casam |
| `"2024-06-14T19:46:33Z"` | String | filtro de intervalo por data compara texto |
| `3000.00` | Double | erro de arredondamento em dinheiro |
| `{ "$numberDecimal": "3000.00" }` | Decimal128 | ✅ é o que o `BigDecimal` do agregado espera |

### O `_class` não é decoração

```java
mongoOperations.insert(mongoDocs, collectionName);
```

Isso insere `org.bson.Document` **cru** — não passa pelo mapeamento do Spring Data, porque não existe objeto de domínio no caminho. É a via rápida, e cobra um preço: o `_class` que o Spring Data normalmente escreveria sozinho precisa estar **escrito à mão no JSON**.

Sem ele, gravar funciona e ler quebra: o Spring Data não sabe em qual classe materializar o documento.

> Esse é o mesmo campo `_class` que aparece nos documentos criados pela API — a diferença é só quem o escreve.

---

## ⚠️ `auto-drop` — o cuidado principal

```java
if (Boolean.TRUE.equals(properties.getAutoDrop())) {
    mongoOperations.getCollection(collectionName).drop();
}
return mongoOperations.insert(mongoDocs, collectionName).size();
```

`auto-drop: true` **apaga a coleção inteira** antes de inserir.

Para estudar, é exatamente o que se quer: toda inicialização parte do mesmo estado conhecido, o teste manual de ontem não contamina o de hoje, e não existe erro de chave duplicada.

O problema é onde essa configuração está hoje: no **`application.yml` default**, o que vale para todo perfil. Qualquer ambiente que suba com essa configuração perde `products` e `categories` no startup.

Diferença de fundo em relação ao Flyway: migration é **incremental e idempotente** — roda uma vez e registra na `flyway_schema_history`. Esta carga é **destrutiva e repetida** — roda toda vez, do zero. São ferramentas para propósitos diferentes; o risco aparece quando a segunda é confundida com a primeira.

Correção sugerida (registrada como pendência, não aplicada):

```yaml
# application.yml — seguro por padrão
algashop:
  data-load:
    enabled: false
    auto-drop: false

# application-development.yml — só aqui liga
algashop:
  data-load:
    enabled: true
    auto-drop: true
```

---

## Tratamento de erro: falhar sem derrubar

```java
try {
    BsonArray array = BsonArray.parse(rawJson);
    ...
} catch (Exception e) {
    log.error("Failed to parse JSON resource {}", e.getMessage(), e);
    return Collections.emptyList();
}
```

JSON malformado loga e devolve lista vazia — a aplicação **sobe mesmo assim**. É a escolha certa aqui: massa de teste quebrada não deve impedir de trabalhar na API.

Contraste com o `@Validated` das properties, que **impede** a subida. A regra por trás: erro de configuração é erro de programação e aparece cedo; erro de dado de teste é acidente e não pode custar o ambiente inteiro.

O resultado fica visível no log:

```
Data load started
db/testdata/categories.json - Imports: 4/4
db/testdata/products.json - Imports: 6/6
```

Se aparecer `0/6`, os documentos foram parseados mas a inserção falhou; se aparecer `0/0`, o parse falhou antes.

---

## Comparativo com o Flyway

| | Flyway (`ordering`, `billing`) | `DataLoader` (`product-catalog`) |
|---|---|---|
| Cria estrutura | sim, o schema | não precisa — Mongo cria no insert |
| Controle de versão | `flyway_schema_history` | nenhum |
| Execução | uma vez por migration | toda inicialização |
| Idempotente | sim | não (por isso o `auto-drop`) |
| Serve para produção | **sim** | **não** |
| Falha ao subir? | sim, se a migration quebrar | não, só loga |

Não são equivalentes. O `DataLoader` cobre a necessidade de desenvolvimento; versionamento de dados em produção continua sem resposta neste serviço.

---

## Verificando

```bash
docker exec -it $(docker ps -qf name=mongodb) \
  mongosh -u root -p algashop --authenticationDatabase admin

use product_catalog
db.products.countDocuments()
db.products.findOne()
db.products.find({ salePrice: { $lt: "$regularPrice" } })   // não funciona — ver $expr
db.products.find({ $expr: { $lt: ["$salePrice", "$regularPrice"] } })
```

A última linha é a consulta que o filtro `hasDiscount` gera — ver [`consultas-mongo-criteria.md`](../02-persistencia/consultas-mongo-criteria.md).

---

## Pendências registradas

- [ ] **`enabled: true` e `auto-drop: true` estão no `application.yml` default.** Deveriam viver num perfil de desenvolvimento; do jeito que está, qualquer ambiente apaga as coleções ao subir.
- [ ] **Log do driver Mongo em `DEBUG` no perfil default** (`org.mongodb.driver.protocol.command`). Útil para ver a query gerada pelos `Criteria`, barulhento demais para ficar ligado sempre.
- [ ] **A carga não valida o que insere.** Documentos vão crus para a coleção, sem passar pelas invariantes do agregado — um produto com `salePrice > regularPrice` no JSON entra sem reclamar, mesmo sendo estado que o `Product` proíbe.
- [ ] **Nenhum índice é criado.** Seria o lugar natural para `createIndex` nos campos filtrados.
- [ ] **Sem teste.** `parseJsonToDocuments` é uma função pura sobre `String` — daria um teste unitário barato, sem Mongo de pé.

---

## Checklist de revisão

- [ ] Sei por que `ApplicationRunner` e não `@PostConstruct`
- [ ] Sei o que `@ConfigurationProperties` + `@Validated` garantem, e quando derrubam a aplicação
- [ ] Sei por que os JSONs usam `$uuid`, `$date` e `$numberDecimal`
- [ ] Sei por que o `_class` precisa estar escrito no arquivo
- [ ] Entendo o risco do `auto-drop` no `application.yml` default
- [ ] Sei explicar por que este mecanismo **não** substitui o Flyway

---

## Referências

- [MongoDB — Extended JSON](https://www.mongodb.com/docs/manual/reference/mongodb-extended-json/)
- [Spring Boot — `ApplicationRunner`](https://docs.spring.io/spring-boot/reference/features/spring-application.html#features.spring-application.command-line-runner)
- [Spring Boot — `@ConfigurationProperties`](https://docs.spring.io/spring-boot/reference/features/external-config.html)
- [`flyway.md`](../02-persistencia/flyway.md) — o equivalente relacional
- [`ambiente-local.md`](./ambiente-local.md) — subir o Mongo e conectar nele
- [`consultas-mongo-criteria.md`](../02-persistencia/consultas-mongo-criteria.md) — o que consultar esses dados
