# Stubs e Contract Tests no AlgaShop

Este documento explica como o AlgaShop usa **Spring Cloud Contract**, **WireMock** e **Stub Runner** para testar a comunicação entre microsserviços sem precisar subir todos eles ao mesmo tempo.

---

## O problema que estamos resolvendo

O `algashop-ordering` precisa chamar o `product-catalog` para buscar dados de um produto ao criar um pedido. Como testar isso de forma confiável?

**Opção ruim:** subir todos os microsserviços toda vez que rodar os testes — lento, frágil, e dependente de infraestrutura.

**Opção boa:** usar um servidor fake (stub) que simula as respostas do `product-catalog` durante os testes.

A questão é: **quem garante que o servidor fake retorna exatamente o que o serviço real retornaria?** É aí que entra o Spring Cloud Contract.

---

## O ecossistema: três peças que se conectam

```
[product-catalog]                    [algashop-ordering]
      |                                       |
 Define contratos               Usa stubs gerados pelos contratos
 Roda testes de provider        Roda testes de integration/consumer
      |                                       |
      +-----------> Maven Local <-------------+
                  (stubs publicados)
```

### 1. Contrato (`.groovy`)

Um contrato é um arquivo que descreve formalmente uma interação HTTP: o que o consumer envia e o que o provider deve responder. Fica no provider (`product-catalog`):

```
src/contractTest/resources/contracts/product/findProductByIdV1.groovy
```

```groovy
// Do projeto: product-catalog
Contract.make {
    request {
        method GET()
        headers { accept 'application/json' }
        url("/api/v1/products/fffe6ec2-7103-48b3-8e4f-3b58e43fb75a")
    }
    response {
        status 200
        headers { contentType 'application/json' }
        body([
            id: fromRequest().path(3),   // pega o ID da URL dinamicamente
            name: 'Notebook X11',
            regularPrice: 1500.00,
            salePrice: 1000.00,
            inStock: true,
            enabled: true,
            // ...
        ])
    }
}
```

Este arquivo serve dois propósitos ao mesmo tempo:

| Para quem | O que gera | Onde roda |
|-----------|-----------|-----------|
| **Provider** (product-catalog) | Testes JUnit gerados automaticamente que verificam se o endpoint real se comporta como descrito | `./gradlew contractTest` no product-catalog |
| **Consumer** (algashop-ordering) | Um stub WireMock (JAR ou JSON) que simula o comportamento descrito | Nos testes de integração do ordering |

---

### 2. Teste de Provider (Contract Test)

Quando o `product-catalog` roda `./gradlew contractTest`, o Spring Cloud Contract:

1. Lê todos os arquivos `.groovy` em `src/contractTest/resources/contracts/`
2. Gera automaticamente classes de teste JUnit
3. Executa esses testes contra o endpoint real

A classe base `ProductBase.java` configura o contexto Spring para esses testes gerados:

```
src/contractTest/java/.../contract/base/ProductBase.java
```

O subdiretório do contrato define qual base usar. Por exemplo:
- Contratos em `contracts/product/` → usa `ProductBase`
- Contratos em `contracts/order/` → usa `OrderBase`

Isso é configurado no `build.gradle`:

```groovy
// algashop-ordering/build.gradle
contracts {
    packageWithBaseClasses = "com.gtech.algashop.contract.base"
    baseClassMappings {
        baseClassMapping("shoppingcart", "com.gtech.algashop.contract.base.ShoppingCartBase")
        baseClassMapping("order", "com.gtech.algashop.contract.base.OrderBase")
    }
}
```

**O que isso garante:** se o `product-catalog` mudar um campo do response e o teste de contrato quebrar, o time sabe antes de publicar uma versão nova que vai quebrar o `ordering`.

---

### 3. Stub publicado no Maven Local

Depois que os contratos passam, o provider publica os stubs:

```bash
./gradlew publishStubsToScm
# ou
./gradlew publishToMavenLocal
```

Isso gera um JAR de stubs com os mapeamentos WireMock dentro:

```
~/.m2/repository/com/algaworks/algashop/product-catalog/0.0.1-SNAPSHOT/
    product-catalog-0.0.1-SNAPSHOT-stubs.jar
        META-INF/
            com.algaworks.algashop/
                product-catalog/
                    mappings/
                        product/
                            findProductByIdV1.json   <-- gerado a partir do .groovy
```

---

## Como o `algashop-ordering` usa os stubs

O projeto usa **duas abordagens** para simular serviços externos nos testes. É importante entender quando usar cada uma.

---

### Abordagem A: WireMock Manual (usado hoje no `OrderControllerIT`)

```java
// OrderControllerIT.java
@BeforeEach
void setup() {
    wireMockProductCatalog = new WireMockServer(
        options()
            .port(8781)
            .usingFilesUnderDirectory("src/test/resources/wiremock/product-catalog")
            .extensions(new ResponseTemplateTransformer(true))
    );
    wireMockProductCatalog.start();
}

@AfterEach
void after() {
    wireMockProductCatalog.stop();
}
```

O WireMock lê os JSONs manualmente escritos em:

```
src/test/resources/wiremock/
    product-catalog/
        mappings/
            get-product-by-id-v1.json        <- produto encontrado (200)
            get-product-by-id-v1-not-found.json  <- produto não encontrado (404)
    rapidex/
        mappings/
            rapidex.json                     <- serviço de frete
```

**Estrutura obrigatória do diretório WireMock:**

```
<raiz>/
    mappings/       <- WireMock procura stubs AQUI (nome fixo)
        *.json
    __files/        <- arquivos de body grandes (opcional)
        *.json
```

O nome `mappings/` é uma convenção do WireMock. Se a pasta não existir com esse nome exato, os stubs não são carregados.

**Exemplo de stub JSON manual** (`get-product-by-id-v1.json`):

```json
{
  "id": "4f229073-2e69-4479-ba24-9a47aa98cfc5",
  "request": {
    "url": "/api/v1/products/fffe6ec2-7103-48b3-8e4f-3b58e43fb75a",
    "method": "GET",
    "headers": {
      "Accept": { "matches": "application/json.*" }
    }
  },
  "response": {
    "status": 200,
    "body": "{\"id\":\"{{{request.path.[3]}}}\",\"name\":\"Notebook X11\",\"salePrice\":1000.00,...}",
    "headers": { "Content-Type": "application/json" },
    "transformers": ["response-template", "spring-cloud-contract"]
  }
}
```

Detalhe importante: `{{{request.path.[3]}}}` é uma sintaxe do **Handlebars** (Response Template Transformer). O índice `[3]` pega o 4º segmento da URL (base 0):

```
/api/v1/products/fffe6ec2-7103-48b3-8e4f-3b58e43fb75a
  [0] [1]   [2]           [3]
```

Isso faz o stub retornar o mesmo ID que foi passado na URL, tornando a resposta dinâmica.

**Quando usar WireMock Manual:**
- Quando o serviço externo **não usa Spring Cloud Contract** (ex: Rapidex, APIs de terceiros)
- Quando você precisa de controle total sobre o stub, incluindo cenários de erro, delays, e comportamentos especiais
- Para simular serviços que você não controla

---

### Abordagem B: Stub Runner (comentado no projeto, mas configurado)

```java
// OrderControllerIT.java (linha 34 - comentado)
// @AutoConfigureStubRunner(
//     stubsMode = StubRunnerProperties.StubsMode.LOCAL,
//     ids = "com.algaworks.algashop:product-catalog:0.0.1-SNAPSHOT:8781"
// )
```

Com `@AutoConfigureStubRunner`, o Spring automaticamente:

1. Baixa o JAR de stubs do `product-catalog` do Maven Local (ou Nexus/Artifactory)
2. Sobe um servidor WireMock na porta `8781` com os stubs do JAR
3. Para o servidor ao final dos testes

O formato do `ids` é: `groupId:artifactId:version:porta`

```
com.algaworks.algashop : product-catalog : 0.0.1-SNAPSHOT : 8781
      groupId              artifactId          version       porta
```

**Quando usar Stub Runner:**
- Quando o serviço externo **usa Spring Cloud Contract** e publica stubs
- Quando você quer **garantia de contrato**: os stubs vêm do mesmo artefato que passou nos testes do provider
- Quando há uma pipeline CI/CD publicando stubs num repositório Maven

**Por que está comentado:** o `product-catalog` precisa ter publicado o JAR de stubs localmente antes do teste rodar. Durante o desenvolvimento, é mais prático usar WireMock manual até o fluxo de publicação estar estabelecido.

---

### Abordagem C: Stub Runner standalone (uso manual/desenvolvimento local)

O arquivo `etc/stub-runner/run-product-catalog-stub.sh` serve para subir um servidor WireMock standalone fora dos testes:

```bash
java -jar stub-runner.jar \
  --stubrunner.ids=com.algaworks.algashop:product-catalog:0.0.1-SNAPSHOT:8083 \
  --stubrunner.stubs-mode=LOCAL
```

**Quando usar o Stub Runner standalone:**
- Quando você está **desenvolvendo** o `algashop-ordering` e quer simular o `product-catalog` sem subí-lo
- Para testar manualmente via Postman/curl apontando para `localhost:8083`
- Para demonstrações ou ambientes de desenvolvimento compartilhado

```
Desenvolvimento local:
  você → POST /api/v1/orders → algashop-ordering (porta 8080)
                                       |
                                       v
                           stub-runner (porta 8083) ← simula product-catalog
```

---

## Comparativo: as três abordagens

| | WireMock Manual | Stub Runner (`@AutoConfigureStubRunner`) | Stub Runner Standalone |
|---|---|---|---|
| **Onde vive** | `src/test/resources/wiremock/` | Anotação na classe de teste | `etc/stub-runner/` |
| **Stubs vêm de** | JSONs escritos manualmente | JAR gerado pelos contratos | JAR gerado pelos contratos |
| **Garantia de contrato** | Nenhuma (pode ficar desatualizado) | Alta (stubs vêm do provider) | Alta (stubs vêm do provider) |
| **Serve para** | APIs de terceiros, cenários especiais | Testes de integração com contrato | Desenvolvimento local manual |
| **Pré-requisito** | Nenhum | JAR publicado no Maven | JAR publicado no Maven |

---

## Fluxo completo no projeto

```
1. PROVIDER define contratos
   product-catalog/src/contractTest/resources/contracts/product/findProductByIdV1.groovy

2. PROVIDER roda testes de contrato
   ./gradlew contractTest   (no product-catalog)
   -> verifica que o endpoint real honra o contrato
   -> falha se a API não bate com o .groovy

3. PROVIDER publica stubs
   ./gradlew publishToMavenLocal
   -> gera product-catalog-0.0.1-SNAPSHOT-stubs.jar em ~/.m2

4a. CONSUMER usa WireMock manual (abordagem atual)
    OrderControllerIT carrega JSONs de src/test/resources/wiremock/product-catalog/mappings/

4b. CONSUMER usa Stub Runner (abordagem com contrato)
    @AutoConfigureStubRunner carrega o JAR do ~/.m2 e sobe WireMock na porta 8781

5. CONSUMER roda testes de integração
   ./gradlew integrationTest   (no algashop-ordering)
   -> OrderControllerIT roda com WireMock simulando o product-catalog
```

---

## Os testes do `OrderControllerIT` explicados

### `shouldAndOrderUsingProduct` e `shouldAndOrderUsingProductUsingDTO`

Cenário feliz. WireMock responde 200 para o produto `fffe6ec2-...` (stub `get-product-by-id-v1.json`). O teste verifica que o pedido é criado com status 201 e persiste no banco.

### `shouldNotCreateOrderWhenCustomerWasNotFound`

O JSON de request usa um `customerId` inválido. O ordering busca o customer no banco e não acha → retorna 422 (Unprocessable Entity). O WireMock do product-catalog nem é chamado.

### `shouldNotCreateOrderWhenProductApiIsUnavailable`

```java
wireMockProductCatalog.stop();  // derruba o servidor antes da chamada
```

O ordering tenta chamar o product-catalog e leva um `ConnectionRefused`. O `ApiExceptionHandler` captura e retorna 504 (Gateway Timeout). Testa o circuit de timeout/unavailability.

### `shouldNotCreateOrderWhenProductWhenProductDoesNotExists`

O JSON usa o productId `21651a12-...`, que tem stub mapeado para 404 (`get-product-by-id-v1-not-found.json`). O ordering recebe 404 do product-catalog e retorna 502 (Bad Gateway) — indicando que o serviço upstream retornou erro.

---

## O contrato define valores para dois contextos diferentes

Nos arquivos `.groovy`, alguns campos usam `value(test(...), stub(...))`:

```groovy
// createOrderWithProductV1.groovy
body([
    productId: value(test(anyUuid()), stub(anyUuid())),
    quantity:  value(test(1),         stub(anyPositiveInt())),
])
```

| Contexto | Usado quando | Significado |
|----------|-------------|------------|
| `test(...)` | Testes gerados no **provider** | O valor que o teste JUnit vai enviar ao endpoint real |
| `stub(...)` | Stub WireMock no **consumer** | O padrão que o WireMock vai aceitar quando o consumer chamar |

Com `test(1)` o teste do provider manda `quantity: 1` fixo. Com `stub(anyPositiveInt())` o stub do consumer aceita qualquer inteiro positivo, tornando os testes de consumer mais flexíveis.

---

## Estrutura de arquivos relacionados no projeto

```
algashop-meta/
    etc/
        stub-runner/
            stub-runner.jar                          <- Stub Runner standalone
            run-product-catalog-stub.sh              <- Script para desenvolvimento local
        wiremock/
            get-product-by-id-v1.json                <- Stubs para uso fora dos testes
            get-product-by-id-v1-not-found.json
            rapidex.json

    microservices/
        product-catalog/
            src/contractTest/resources/contracts/
                product/
                    findProductByIdV1.groovy          <- Contrato: buscar produto
                    findProductByIdV1NotFound.groovy  <- Contrato: produto não encontrado
                    createProductV1.groovy
                    ...

        algashop-ordering/
            src/
                contractTest/resources/contracts/    <- Contratos do ordering como provider
                    order/
                        createOrderWithProductV1.groovy
                    shoppingcart/
                        createShoppingCartV1.groovy

                test/
                    java/.../presentation/order/
                        OrderControllerIT.java        <- Testes de integração (consumer)
                    resources/
                        wiremock/
                            product-catalog/
                                mappings/
                                    get-product-by-id-v1.json
                                    get-product-by-id-v1-not-found.json
                            rapidex/
                                mappings/
                                    rapidex.json
```

---

## Separando unidade, contrato e integração no Gradle

Com três tipos de teste convivendo no `product-catalog`, deixar tudo em `./gradlew test` cria um problema prático: o desenvolvedor que só quer validar uma regra de negócio precisa de MongoDB de pé.

A separação por **suíte**:

| Task | Roda o quê | Precisa de infra? |
|---|---|---|
| `test` | unitários e testes de fatia | não |
| `contractTest` | gerados pelo Spring Cloud Contract a partir dos `.groovy` | não (o provider é mockado) |
| `integrationTest` | classes com sufixo `*IT` | **sim** (Mongo) |

```groovy
// build.gradle
tasks.named('check') {
    dependsOn(test, contractTest, integrationTest)
}

test {
    filter {
        excludeTestsMatching("*IT")   // o dia a dia roda sem container nenhum
    }
}

tasks.register('integrationTest', Test) {
    testClassesDirs = sourceSets.test.output.classesDirs
    classpath = sourceSets.test.runtimeClasspath

    shouldRunAfter test

    filter {
        includeTestsMatching "*IT"
        excludeTestsMatching "*Test"
    }
}
```

Três decisões dentro desse trecho:

1. **Sem source set próprio.** `src/integrationTest/java` seria a alternativa "oficial", mas exigiria configurar dependências e classpath separados. Aqui o mesmo `src/test` serve às duas tasks, e o filtro por sufixo faz a divisão. Menos cerimônia; a convenção de nome (`*IT` vs. `*Test`) vira parte do contrato do projeto.
2. **`shouldRunAfter`, não `dependsOn`.** É ordem, não dependência: se as duas rodarem, os unitários vêm primeiro — falha barata aparece antes de gastar tempo com a suíte lenta. Mas `integrationTest` continua podendo rodar sozinho.
3. **`check` amarra as três.** Rodar solto é opcional; `./gradlew check` (e por consequência `build`) valida tudo.

### O javaagent do Mockito

```groovy
configurations {
    mockitoAgent { transitive = false }
}
dependencies {
    mockitoAgent 'org.mockito:mockito-core'
}
test {
    jvmArgs += "-javaagent:${configurations.mockitoAgent.asPath}"
}
```

O Mockito instrumenta bytecode em tempo de execução, e até o JDK 20 fazia isso se auto-anexando à JVM. A partir do JDK 21 esse *self-attach* passa a emitir aviso, e o plano é falhar em versões seguintes. Passar o JAR explicitamente como `-javaagent` é o caminho suportado.

O `transitive = false` é porque a configuração existe só para **resolver o caminho de um arquivo** — não se quer a árvore de dependências junto, só o JAR do `mockito-core`.

### Fatia de contexto: `@DataMongoTest`

O `ProductRepositoryIT` é o exemplo mais simples de teste de integração no serviço:

```java
@DataMongoTest              // sobe só a camada de persistência — sem controllers, sem services
@Import(MongoConfig.class)  // a fatia NÃO carrega @Configuration da aplicação
class ProductRepositoryIT {

    @Autowired
    private ProductRepository productRepository;
}
```

O `@Import` não é opcional. As anotações de fatia (`@DataMongoTest`, `@WebMvcTest`, `@DataJpaTest`) carregam apenas as autoconfigurações relevantes — as `@Configuration` da aplicação ficam de fora de propósito, para o contexto ser pequeno. Sem importar o `MongoConfig`, faltariam o `UuidRepresentation.STANDARD` e os conversores de `OffsetDateTime`, e o teste quebraria na leitura dos documentos.

Comparar com o `@WebMvcTest` que o `ProductBase` usa para os contract tests: mesma ideia, camada oposta — lá sobe só a web e o service é mockado.

> Os dados que esse teste lê vêm do `DataLoader` — ver [`carga-de-dados-mongo.md`](../04-infraestrutura/carga-de-dados-mongo.md).

---

## Resumo mental

> **Contrato** = acordo escrito em `.groovy` que vive no **provider**.
>
> **Contract Test** = teste gerado automaticamente que garante que o **provider honra o contrato**.
>
> **Stub** = servidor fake gerado a partir do contrato que o **consumer usa nos seus testes**.
>
> **WireMock Manual** = stub escrito à mão em JSON, sem garantia de contrato — útil para APIs de terceiros.
>
> **Stub Runner** = mecanismo que baixa e sobe automaticamente os stubs gerados pelo contrato — garante que consumer e provider estão sincronizados.
