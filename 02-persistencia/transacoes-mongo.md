# Transações e Replica Set no MongoDB

> Como duas escritas em coleções diferentes passaram a cair juntas: por que transação no Mongo exige um replica set, o que o `MongoTransactionManager` liga, e quando transação **não** acrescenta nada.
> Código real: `application/product/management/ProductManagementApplicationService.java`, `infrastructure/persistence/MongoConfig.java`, `domain/product/StockMovement.java`, `docker-compose.tools.yml`.

> Este documento fecha uma armadilha registrada na Fase 13: *"Não há transação. O evento é publicado depois do ajuste; se algo falhar entre uma coisa e outra, o estoque mudou e ninguém foi avisado."*

---

## O problema

A Fase 13 deixou o ajuste de estoque atômico e correto. O que ela **não** tinha era memória: o saldo dizia `40`, e nada no banco explicava como ele saiu de `50`.

A resposta é uma segunda coleção, `stock_movements`, com uma linha por entrada ou saída. E aí nasce um problema que não existia:

```java
// products: o findAndModify atômico da Fase 13
StockMovement movement = stockService.withdraw(product, quantity);

// stock_movements: uma segunda escrita, em outra coleção
stockMovementRepository.save(movement);
```

Cada uma dessas escritas é atômica — o Mongo garante isso para **um** documento. O que ninguém garante é que **as duas** aconteçam. Se a segunda falhar, o estoque já baixou e o movimento se perdeu.

E repare no formato dessa falha: ela é silenciosa. Não há saldo negativo, não há número errado, nada dispara alarme. Sobra um estoque perfeitamente correto que **nenhum histórico explica** — e isso é pior que um saldo errado, porque com o saldo errado você ao menos sabe que existe um problema. Aqui você só descobre quando precisa auditar, e a essa altura não há como reconstruir o que aconteceu.

---

## Por que isso arrastou o `docker-compose` junto

A resposta natural é `@Transactional`. Só que no MongoDB a transação de múltiplos documentos tem um pré-requisito de infraestrutura:

> **Transação no MongoDB só existe dentro de um replica set** (ou de um cluster com `mongos`). Numa instância única, ela não é lenta nem limitada — ela simplesmente não existe.

O motivo é o *oplog*. A transação do Mongo é construída sobre o mesmo mecanismo que replica dados entre os nós: é lá que a intenção fica registrada antes de virar efeito visível. Sem replica set não há oplog nesse sentido, e portanto não há onde escrever "estou no meio de algo".

Tentar sem isso dá um erro bem específico — vale reconhecê-lo:

```
Command failed with error 20 (IllegalOperation):
'Transaction numbers are only allowed on a replica set member or mongos'
```

Por isso uma decisão de **domínio** ("quero histórico de movimentação") acabou mudando o `docker-compose.tools.yml`. É um caso raro e instrutivo de requisito funcional atravessando até a infraestrutura.

### O cluster local

```yaml
# docker-compose.tools.yml
algashop-mongodb-1:
  image: mongo:8
  command: ["mongod", "--replSet", "rs0", "--bind_ip_all"]
  ports:
    - 27017:27017
# ...mongodb-2 na 27018 e mongodb-3 na 27019, idênticos
```

Três nós, e um quarto serviço efêmero que só existe para configurá-los:

```yaml
algashop-mongodb-init:
  image: mongo:8
  depends_on:
    algashop-mongodb-1: { condition: service_healthy }
    # ...os outros dois, também service_healthy
  command: >
    mongosh --host algashop-mongodb-1 --eval '
      rs.initiate({
        _id: "rs0",
        members: [
          { _id: 0, host: "algashop-mongodb-1:27017", priority: 2},
          { _id: 1, host: "algashop-mongodb-2:27017", priority: 0},
          { _id: 2, host: "algashop-mongodb-3:27017", priority: 0}
        ]
      })
    ' || true;
```

Três detalhes que parecem acessórios e não são:

**O `|| true`** torna o serviço idempotente. `rs.initiate` num replica set já iniciado retorna erro, e sem isso todo `docker compose up` subsequente terminaria em falha. Com ele, a segunda execução é inofensiva.

**O `condition: service_healthy`** evita a corrida óbvia: `rs.initiate` contra um `mongod` que ainda está subindo falha, e aí o `|| true` engoliria a falha de verdade.

**As prioridades — `2` no primeiro nó, `0` nos outros dois.** Prioridade zero significa *"este nó nunca pode ser eleito primário"*. Ou seja: **não há failover, de propósito**. Num cluster de produção isso seria um erro grave; aqui é o contrário. Num ambiente de estudo, eleição não determinística troca um problema por outro — você quer saber, sem pensar, que o primário está na `27017`. O objetivo do cluster aqui não é disponibilidade, é **habilitar transação**, que é o requisito mínimo indivisível.

> ⚠️ **O cluster subiu sem autenticação.** As variáveis `MONGO_INITDB_ROOT_*` saíram junto com o nó único, e a URI da aplicação perdeu o `root:algashop@` e o `?authSource=admin`. É aceitável num ambiente local e **não** é o que se faz fora dele — com auth ligada, replica set exige ainda um `--keyFile` compartilhado entre os nós para a autenticação interna. Essa simplificação cobrou o seu preço; ver [Como um teste ficou órfão](#como-um-teste-ficou-órfão).

Os três nós anunciam-se pelos nomes `algashop-mongodb-N`, que só existem dentro da rede do Docker. Como a aplicação roda **fora** dela, o driver precisa resolver esses nomes — daí o `etc/hostnames/`, com as entradas para o arquivo `hosts` e o passo a passo por sistema operacional. Ver [`ambiente-local.md`](../04-infraestrutura/ambiente-local.md).

---

## O bean que liga tudo

```java
// infrastructure/persistence/MongoConfig.java
@Bean
public MongoTransactionManager mongoTransactionManager(MongoDatabaseFactory factory) {
    return new MongoTransactionManager(factory);
}
```

Cinco linhas, e sem elas nada do resto funciona. O detalhe que vale gravar:

> ⚠️ **Sem esse bean, `@Transactional` não falha — ele não faz nada.** Nenhum erro, nenhum aviso, nenhum log. A anotação continua lá, legível, sugerindo uma garantia que não existe. É a pior categoria de defeito: o que se parece exatamente com o funcionamento correto.

E há uma segunda ausência que confunde na primeira leitura: **`@EnableTransactionManagement` não aparece em lugar nenhum do projeto.** Não é esquecimento. O Spring Boot liga o proxy transacional sozinho assim que existe um `PlatformTransactionManager` no contexto. A habilitação é implícita — ótimo até o dia em que alguém procura onde foi ligada e não encontra.

---

## Onde a transação entra, e onde não entra

```java
@Transactional
public void withdraw(UUID productId, int quantity) {
    Product product = findProduct(productId);
    StockMovement movement = stockService.withdraw(product, quantity);
    stockMovementRepository.save(movement);
}
```

Só `restock` e `withdraw` são transacionais. `create`, `update`, `disable` e `enable` **não são** — e isso é decisão, não omissão:

| Método | Escreve em | Transação |
|---|---|---|
| `create`, `update`, `disable`, `enable` | um documento de `products` | não acrescentaria nada |
| `restock`, `withdraw` | `products` **e** `stock_movements` | é o que faz as duas caírem juntas |

A regra é direta: **escrita de um único documento no MongoDB já é atômica por natureza.** Envolver isso numa transação adiciona custo — sessão, oplog, coordenação — em troca de uma garantia que o banco já dava de graça. Transação não é selo de qualidade que se aplica por precaução; é ferramenta para um problema específico, que é ter **mais de uma** escrita que precisa ser indivisível.

### O adaptador não mudou uma linha

O detalhe mais elegante desta fase: o `QuantityInStockAdjustmentMongoDBImpl`, escrito na Fase 13 sem nenhuma noção de transação, passou a participar dela **sem alteração**.

Isso acontece porque o `MongoTransactionManager` amarra a sessão à *thread* corrente, e o `MongoOperations` injetado no adaptador procura por essa sessão antes de cada operação. Se existe uma, o `findAndModify` entra nela; se não existe, ele roda solto como antes.

É o dividendo de ter dependido de uma abstração (`MongoOperations`) em vez do driver cru — o comportamento novo chegou por baixo, sem tocar em quem o usa.

### Os eventos entraram no commit

Um efeito de segunda ordem, e o mais fácil de não perceber. Os listeners de estoque são **síncronos**, então rodam dentro da transação de quem publicou:

```
withdraw()  →  findAndModify        ─┐
            →  publish(SoldOut)      │  tudo dentro
            →  listener roda aqui    │  da mesma transação
            →  save(movement)       ─┘
                                     → commit
```

Uma exceção no listener desfaz o ajuste de estoque junto. O evento deixou de ser um "depois" e virou parte do mesmo commit.

> ⚠️ **É uma garantia de uma anotação de distância do fim.** Um `@Async` em qualquer handler de estoque o tiraria do rollback: outra thread significa fora da sessão transacional, e ele passaria a ler o estado **anterior** ao commit. Nada quebraria, nada avisaria. E o precedente já existe no arquivo — o handler de `ProductPriceChangedEvent` ganhou `@Async` nesta mesma leva (hoje inofensivo, porque `update()` não é transacional). Ver [`eventos-e-listeners.md`](../01-arquitetura-design/eventos-e-listeners.md).

---

## O registro de movimentação

```java
@Document(collection = "stock_movements")
public class StockMovement {
    @EqualsAndHashCode.Include private UUID id;
    private OffsetDateTime occurredAt;
    @Indexed private UUID productId;
    private Integer movementQuantity;
    private Integer previousQuantity;
    private Integer newQuantity;
    private MovementType type;   // STOCK_IN | STOCK_OUT
}
```

O contraste com o `Product`, que mora no mesmo pacote, é o que vale estudar aqui:

| | `Product` | `StockMovement` |
|---|---|---|
| `AbstractAggregateRoot` | sim — registra eventos | **não** |
| `@Version` | sim — lock otimista | **não** |
| Auditoria (`@CreatedDate`…) | sim | **não** — `occurredAt` é o dado, não metadado |
| Alterado depois de criado | sim | **nunca** |

Nada disso é economia de esforço: **um registro imutável de fato consumado não tem invariante para proteger.** Toda a maquinaria do agregado — versão, eventos, guardas em setters — existe para defender regra sobre estado que muda. Aqui não há estado que mude. Aplicar o padrão do `Product` por simetria seria cerimônia sem função.

Duas escolhas menores, com razão:

**`movementQuantity` é sempre positivo** — o sinal vive no `type`. Guardar `-10` para uma saída economiza um campo e cobra na leitura: toda soma passa a depender de ninguém ter errado o sinal na escrita. Com o `type` explícito, *"quanto saiu no mês"* é um filtro, não uma convenção que alguém precisa lembrar.

**`@Indexed` no `productId`** — a consulta óbvia desta coleção é "o histórico deste produto", e ela é a única coleção do serviço que **só cresce**: nada é apagado, nada é atualizado. Sem índice, essa consulta piora todo dia.

---

## Como um teste ficou órfão

Vale registrar, porque o estrago não veio de uma decisão errada — veio de uma decisão certa cujo alcance passou despercebido.

Ao virar cluster, o Mongo perdeu a autenticação. A URI de desenvolvimento foi atualizada. A de **teste** não:

```yaml
# src/test/resources/application-test-env.yml — como estava
uri: mongodb://root:algashop@localhost:27017/product_catalog_test?authSource=admin
```

Dois dos três testes de integração não notaram, porque haviam ganhado Testcontainers e recebem a URI do container. O terceiro — o `QuantityInStockAdjustmentConcurrencyIT`, justamente **o teste que sustenta a Fase 13 inteira** — ficou de fora dessa migração e caiu nessa URI:

```
Command failed with error 18 (AuthenticationFailed): 'Authentication failed.'
```

Os três testes de concorrência morriam na subida do contexto. E ninguém viu, porque a suíte `integrationTest` não chegou a ser executada depois da mudança de infraestrutura — os `build/test-results/` traziam `test` e `contractTest`, nunca `integrationTest`.

A lição não é sobre Mongo: **quando a infraestrutura muda, a suíte que depende dela precisa rodar antes de o trabalho ser considerado pronto.** Uma mudança de compose não parece capaz de apagar uma garantia de concorrência escrita duas semanas antes — e foi exatamente o que fez.

> Uma hipótese que a verificação **derrubou**, e que vale registrar por honestidade: o `TestContainerMongoDBConfig` inicializava o replica set anunciando `localhost:27017`, que do lado da máquina é o primário do cluster de desenvolvimento — e o nome do conjunto (`rs0`) também coincidia. A suspeita era que a suíte pudesse estar escrevendo no banco de desenvolvimento. Comparados os documentos antes e depois de uma execução completa, o `product_catalog` ficou **intacto**: o driver permaneceu na porta mapeada do container. O risco era real de se construir; o dano não aconteceu.

---

## Provando que funciona

Teste de transação tem um jeito específico de passar por engano: se a anotação não estiver valendo, os cenários de caminho feliz continuam verdes e só o rollback mente. Por isso o `StockTransactionIT` é um `@SpringBootTest`, e não uma fatia:

> Uma fatia como `@DataMongoTest` não carrega o proxy transacional. O `@Transactional` viraria enfeite, o teste passaria — e passaria pelo motivo errado, que é a pior forma de passar.

```java
@Test
void shouldRollbackTheStockAdjustmentWhenTheMovementFails() {
    doThrow(new DataAccessResourceFailureException("stock_movements indisponivel"))
            .when(stockMovementRepository).save(any(StockMovement.class));

    assertThatExceptionOfType(DataAccessResourceFailureException.class)
            .isThrownBy(() -> productManagementApplicationService.withdraw(EXISTING_PRODUCT, 10));

    // as duas afirmações são o teste: nem meia operação ficou
    assertThat(stockOf(EXISTING_PRODUCT)).isEqualTo(INITIAL_STOCK);
    assertThat(mongoOperations.count(new Query(), StockMovement.class)).isZero();
}
```

O `@MockitoSpyBean` é o que permite os dois cenários na mesma classe: por padrão ele delega para o repositório real, então o caminho feliz grava de verdade; só o teste de rollback troca o comportamento, e só do `save`.

**E a prova de que o teste tem dentes** — a mesma disciplina do teste de concorrência da fase anterior. Comentando as duas anotações `@Transactional`:

```
shouldRollbackTheStockAdjustmentWhenTheMovementFails()  expected: 50  but was: 40
shouldRollbackTheRestockWhenTheMovementFails()          expected: 50  but was: 75
```

O estoque mudou e o movimento nunca foi escrito — exatamente o saldo órfão descrito lá em cima. Restaurada a anotação, os quatro voltam ao verde. Um teste de rollback que não foi visto falhando não prova coisa alguma.

---

## Armadilhas

1. **Sem `MongoTransactionManager`, `@Transactional` é decoração.** Não avisa, não falha, não loga. Só não protege.
2. **Sem replica set, nem sobe** — `error 20 (IllegalOperation)`. Este pelo menos é barulhento.
3. **`@Async` num listener o tira do rollback.** Outra thread, outra sessão; ele lê o estado anterior ao commit e sobrevive à falha que deveria desfazê-lo.
4. **Transação em escrita de um documento só é custo sem benefício.** O Mongo já garante atomicidade por documento.
5. **Fatia de teste não carrega o proxy transacional.** O teste passa sem que a transação exista.
6. **Mudar a infraestrutura invalida configuração de teste em silêncio.** Foi assim que três testes de concorrência morreram sem ninguém perceber.
7. **`rs.initiate` sem `|| true` quebra o segundo `up`.** Idempotência aqui não é elegância, é o compose voltar a subir amanhã.
8. **`stock_movements` não é gerenciada pelo `DataLoader`.** As `sources` do YAML só listam `products` e `categories`, então o `auto-drop` não a alcança: os movimentos **acumulam** entre execuções, e um teste que dependa de contagem precisa limpá-la à mão.

---

## Pendências registradas

- [ ] **Sem `writeConcern` e `readConcern` explícitos na URI.** Funciona pelo default de `majority` num conjunto de três nós — que é o que a transação exige para o commit ser durável —, mas depender de default numa garantia dessas é frágil: bastaria alguém subir o cluster com dois nós.
- [ ] **Os perfis `docker-env` e `production-env` não existem.** O `application.yml` referencia os dois em `spring.profiles.group`, e nenhum arquivo correspondente foi criado. Perfil sem arquivo é ignorado em silêncio, então rodar em `production` cairia no default `mongodb://localhost/test` — sem URI nenhuma.
- [ ] **Nada valida que o contexto completo sobe.** O `ProductCatalogApplicationTests` (o `contextLoads`) foi removido nesta leva e era o único `@SpringBootTest` da suíte; hoje o `StockTransactionIT` cobre isso de lado, por acidente, não por desenho.
- [ ] **O cluster local não tem autenticação.** Aceitável aqui, e a distância para um ambiente real inclui `--keyFile` entre os nós, usuários por banco e TLS.
- [ ] **Sem contrato para `/restock` e `/withdraw`.** Herdada da Fase 13 e ainda aberta.
- [ ] **A propagação continua sem outbox.** A transação resolve as duas escritas *deste* serviço; não resolve o evento que deveria sair para fora dele.
- [x] ~~**Publicação de evento sem transação.**~~ Resolvido: os listeners de estoque são síncronos e rodam dentro do commit.
- [x] ~~**Estoque podia mudar sem registro que o explicasse.**~~ Resolvido: `stock_movements` e o `@Transactional` que a amarra ao ajuste.

---

## Checklist de revisão

- [ ] Sei por que transação no MongoDB exige replica set, e o que o oplog tem a ver com isso
- [ ] Sei reconhecer o `error 20 (IllegalOperation)` e o que ele significa
- [ ] Entendo por que `MongoTransactionManager` ausente é pior que erro de compilação
- [ ] Sei dizer por que `create`/`update` não são transacionais e `withdraw` é
- [ ] Entendo como o `findAndModify` entrou na transação sem mudar de código
- [ ] Sei o que um `@Async` num listener faria com a garantia de rollback
- [ ] Sei por que `StockMovement` não tem `@Version` nem estende `AbstractAggregateRoot`
- [ ] Entendo por que `movementQuantity` é positivo nos dois sentidos
- [ ] Sei por que o teste de transação precisa de `@SpringBootTest` e não de uma fatia
- [ ] Sei por que um teste de rollback que nunca foi visto falhando não prova nada
- [ ] Entendo por que as prioridades do replica set local desligam o failover de propósito

---

## Referências

- [MongoDB — Transactions](https://www.mongodb.com/docs/manual/core/transactions/)
- [MongoDB — Replication](https://www.mongodb.com/docs/manual/replication/)
- [MongoDB — `rs.initiate()`](https://www.mongodb.com/docs/manual/reference/method/rs.initiate/)
- [Spring Data MongoDB — Transactions](https://docs.spring.io/spring-data/mongodb/reference/mongodb/transactions.html)
- [Testcontainers — MongoDB Module](https://java.testcontainers.org/modules/databases/mongodb/)
- [`concorrencia-e-atomicidade.md`](./concorrencia-e-atomicidade.md) — o ajuste atômico que esta fase envolveu numa transação
- [`eventos-e-listeners.md`](../01-arquitetura-design/eventos-e-listeners.md) — os eventos que passaram a rodar dentro do commit
- [`product-catalog-mongo.md`](./product-catalog-mongo.md) — a modelagem do `Product`, com tudo que o `StockMovement` deliberadamente não tem
- [`ambiente-local.md`](../04-infraestrutura/ambiente-local.md) — subir o cluster, o arquivo `hosts` e os problemas comuns
