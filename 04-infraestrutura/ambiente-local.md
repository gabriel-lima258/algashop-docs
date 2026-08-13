# Ambiente local

> Do `git clone` até os serviços rodando. Portas, bancos, ferramentas de apoio e os problemas mais comuns.

---

## 1. Clonar o projeto (com submódulos)

O `algashop-meta` **não contém código** — ele agrega seis repositórios independentes como submódulos Git:

```
algashop-meta/
├── docs/                              → algashop-docs
├── template/                          → algashop-template-inicial
└── microservices/
    ├── algashop-ordering/             → algashop-ordering
    ├── algashop-billing/              → algashop-billing
    ├── billing-scheduler/             → algashop-billing-scheduler
    └── product-catalog/               → algashop-product-catalog
```

Clonar tudo de uma vez:

```bash
git clone --recurse-submodules https://github.com/gabriel-lima258/algashop-meta.git
```

Se já clonou sem os submódulos (as pastas aparecem vazias):

```bash
git submodule update --init --recursive
```

### Trabalhando no dia a dia

| Situação | Comando |
|---|---|
| Puxar as atualizações de todos os submódulos | `git submodule update --remote --merge` |
| Ver o estado de todos de uma vez | `git submodule foreach 'git status --short'` |
| Adicionar um novo submódulo | `git submodule add <url> microservices/<nome>` |
| Fixar o submódulo numa branch | `git submodule set-branch -b main microservices/<nome>` |

> ⚠️ **Cuidado com `git submodule update`.** Sem `--remote`, ele joga o submódulo no commit registrado no meta e **descarta trabalho local não commitado**. Antes de rodar, sempre confira com `git submodule foreach 'git status --short'`.

**Como o versionamento funciona:** o repositório meta guarda apenas um **ponteiro** (o SHA do commit) de cada submódulo. Ao alterar código de um serviço, o fluxo é sempre em duas etapas — commitar dentro do submódulo, depois commitar no meta o ponteiro atualizado:

```bash
cd microservices/product-catalog
git commit -m "feat: ..." && git push

cd ../..
git add microservices/product-catalog
git commit -m "chore: atualizar submodulo product-catalog" && git push
```

Esquecer a segunda etapa é o erro mais comum: o código está no GitHub, mas quem clonar o meta continua recebendo a versão antiga.

---

## 2. Subir a infraestrutura

Os arquivos de composição estão na raiz do meta e são dois, com papéis distintos:

| Arquivo | Contém |
|---|---|
| `docker-compose.tools.yml` | Bancos e serviços de apoio — **é o que você usa no dia a dia** |
| `docker-compose.services.yml` | Os microsserviços empacotados (inclui o `tools` via `include:`) |

### Cenário A — desenvolvimento (o mais comum)

Sobe só a infraestrutura; os serviços você roda pela IDE ou por `./gradlew bootRun`:

```bash
docker compose -f docker-compose.tools.yml up -d
```

> ⚠️ **O `.env` na raiz do meta não é opcional.** É de lá que o Compose lê `REDIS_PASSWORD` para montar o `--requirepass` do Redis. Sem o arquivo, a variável resolve para string vazia, o Redis sobe **sem autenticação**, e as aplicações — que mandam senha — são recusadas. O sintoma é traiçoeiro: tudo responde normalmente, e o cache simplesmente nunca funciona. Ver [`redis.md`](./redis.md).

### Cenário B — tudo em container

Desde a Fase 18 o comando é o mais curto possível:

```bash
docker compose up -d
```

O `docker-compose.yml` inclui o de serviços, que por sua vez inclui o de tools — um `up` sobe **tudo**: infraestrutura, `ordering` (8081), `billing` (8082), `product-catalog` (8083) e o `billing-scheduler`. Exige que as imagens já existam localmente (`gabriel58221/*:dev`) — ver [`docker.md`](./docker.md).

O `billing-scheduler` entrou como o que ele é: um job efêmero. Ele sobe, cancela as faturas vencidas e **encerra com código 0** — por isso `restart: no` e nenhuma porta publicada. Vê-lo como `Exited (0)` no `compose ps` é o comportamento correto, não uma falha. Em produção quem o reexecuta seria um CronJob; aqui, um `docker compose up algashop-billing-scheduler` quando quiser.

Para subir só a infraestrutura, sem os quatro serviços:

```bash
docker compose -f docker-compose.tools.yml up -d
```

> 🔬 **O `ordering` sobe com o Tomcat limitado a 10 threads de propósito**, para que o teste de carga encontre um gargalo numa máquina de desenvolvimento. Para virar a chave das threads virtuais sem editar o compose:
>
> ```bash
> VIRTUAL_THREADS=true docker compose up -d --force-recreate algashop-ordering
> ```
>
> Cuidado: sob carga alta, essa configuração **trava o serviço de forma permanente** neste projeto. O porquê está em [`threads-e-concorrencia.md`](./threads-e-concorrencia.md).

### Comandos úteis

```bash
docker compose -f docker-compose.tools.yml ps         # o que está de pé
docker compose -f docker-compose.tools.yml logs algashop-mongodb-init   # o rs.initiate
docker compose -f docker-compose.tools.yml logs -f algashop-mongodb-1
docker compose -f docker-compose.tools.yml down       # parar (mantém os dados)
docker compose -f docker-compose.tools.yml down -v    # parar E APAGAR os volumes
```

---

## 3. Mapa de portas

A tabela que evita 90% dos problemas de "não conecta":

| Serviço | Host | Container | Observação |
|---|---|---|---|
| `algashop-ordering` | **8081** | 8081 | PostgreSQL |
| `algashop-billing` | **8082** | 8082 | PostgreSQL |
| `product-catalog` | **8083** | 8083 | MongoDB |
| `billing-scheduler` | — | — | sem porta HTTP (só jobs) |
| PostgreSQL | **5433** | 5432 | ⚠️ deslocada de propósito |
| MongoDB nó 1 | **27017** | 27017 | primário — `priority: 2` |
| MongoDB nó 2 | **27018** | 27017 | secundário — `priority: 0` |
| MongoDB nó 3 | **27019** | 27017 | secundário — `priority: 0` |
| Redis | **6379** | 6379 | cache do catálogo (db 0) e do `ordering` (db 1) |
| WireMock | **8787** | 8080 | mock de APIs externas |
| FastPay | **9995** | 9995 | gateway de pagamento simulado |
| LocalStack | **4566** | 4566 | a AWS emulada — só S3, bucket `algashop-product-image` |

O `authorization-server` roda em **9000** e **não está no compose** — sobe por `./gradlew bootRun`. Os outros três apontam o `issuer-uri` para `http://algashop-authorization-server:9000` e é de lá que buscam as chaves públicas.

> Desde a Fase 22 o `ordering` **sobe sem o authorization server no ar**: o lado client passou a declarar `token-uri` em vez de `issuer-uri`, eliminando a descoberta que acontecia durante o refresh do contexto. A primeira ida à rede virou a primeira requisição que precisa de token, não a inicialização. Ver [`oauth2-client-e-token.md`](../05-seguranca/oauth2-client-e-token.md). A porta era 8081 e foi trocada na Fase 20 justamente porque colidia com a do `ordering`. Ver [`authorization-server.md`](../05-seguranca/authorization-server.md).

> Desde a Fase 23 o Postgres hospeda **quatro** bancos de aplicação: `ordering`, `billing`, `authorization_server` e os `*_test`.

> ⚠️ **A porta 5433 não é engano.** O Postgres do projeto é exposto em `5433` no host justamente para não conflitar com uma instalação nativa de PostgreSQL, que ocupa a `5432`. Dentro da rede Docker os containers continuam falando na `5432`.
>
> Por isso as URLs mudam conforme de onde você conecta:
> - Da sua máquina (perfil `development`): `jdbc:postgresql://localhost:5433/ordering`
> - De dentro de um container (perfil `docker`): `jdbc:postgresql://algashop-postgres:5432/ordering`

O mesmo raciocínio vale para o WireMock: `http://localhost:8787` de fora, `http://wiremock:8080` de dentro.

O **LocalStack é a exceção**, e vale entender por quê: a URL pré-assinada é gerada com o endereço que o servidor conhece (`algashop-localstack:4566`) e depois usada **pelo navegador**. Não há tradução possível — o mesmo nome tem que valer dos dois lados, e é isso que as entradas no `hosts` compram.

### Conferindo se um serviço está saudável

Desde a Fase 17 os três serviços com HTTP expõem Actuator:

```bash
curl -s localhost:8081/actuator/health | jq            # tudo: banco, cache, circuitos
curl -s localhost:8081/actuator/health/readiness | jq  # só o essencial para atender
curl -s localhost:8081/actuator/info | jq
```

O `readiness` inclui **só o banco** — cache e circuitos fora do ar não tiram a instância de rotação. Ver [`health-checks.md`](./health-checks.md).

> ⚠️ **Um `/actuator/health` com status `DEGRADED` ainda devolve HTTP 200.** Se você estiver testando com `curl -o /dev/null -w "%{http_code}"`, vai ver 200 mesmo com um circuito aberto. Olhe o corpo.

---

## 4. Os bancos

### PostgreSQL — `ordering` e `billing`

O script `etc/postgres/init-user-db.sh` roda **uma única vez**, na primeira criação do volume, e cria cinco bancos:

```bash
CREATE DATABASE fastpay;
CREATE DATABASE ordering;
CREATE DATABASE ordering_test;      # isolado, para testes de integração
CREATE DATABASE billing;
CREATE DATABASE billing_test;       # idem
```

Credenciais: `postgres` / `postgres`.

> ⚠️ **Alterou o script depois de já ter subido?** Ele não roda de novo — o Docker só executa o `docker-entrypoint-initdb.d` quando o diretório de dados está vazio. Para reprocessar: `docker compose -f docker-compose.tools.yml down -v` (isso **apaga todos os dados**) e suba novamente.

Bancos `*_test` separados existem para os testes de integração não pisarem nos dados de desenvolvimento. O schema é gerenciado por Flyway — ver [`flyway.md`](../02-persistencia/flyway.md).

```bash
psql -h localhost -p 5433 -U postgres -d ordering
```

### MongoDB — `product-catalog`

Desde a Fase 14 **não é uma instância só: são três nós formando o replica set `rs0`.** A razão é uma só, e não é disponibilidade — **transação no MongoDB não existe fora de um replica set**, e o `product-catalog` passou a precisar de uma. Ver [`transacoes-mongo.md`](../02-persistencia/transacoes-mongo.md).

```yaml
algashop-mongodb-1:                # e -2 na 27018, e -3 na 27019
  image: mongo:8
  command: ["mongod", "--replSet", "rs0", "--bind_ip_all"]
  ports:
    - 27017:27017
  healthcheck:
    test: [ "CMD", "mongosh", "--eval", "db.adminCommand('ping')" ]

algashop-mongodb-init:             # efêmero: configura o conjunto e morre
  image: mongo:8
  depends_on:                      # os três precisam estar healthy antes
    algashop-mongodb-1: { condition: service_healthy }
  command: >
    mongosh --host algashop-mongodb-1 --eval 'rs.initiate({...})' || true;
```

O nó 1 tem `priority: 2` e os outros dois têm `priority: 0` — ou seja, **não há failover, de propósito**: o primário é sempre a `27017`. Num cluster de produção seria erro; aqui é o que torna o ambiente previsível.

> ⚠️ **A autenticação foi removida junto com o nó único.** As variáveis `MONGO_INITDB_ROOT_*` saíram, e com elas o `-u root -p algashop` e o `?authSource=admin`. Comandos e URIs antigos com credenciais agora recebem `Authentication failed`.

#### O arquivo `hosts` — passo obrigatório

Os três nós se anunciam no replica set pelos nomes `algashop-mongodb-1/2/3`, que só existem dentro da rede do Docker. Como a aplicação roda **fora** dela, sua máquina precisa saber resolver esses nomes:

```
127.0.0.1       algashop-mongodb-1
127.0.0.1       algashop-mongodb-2
127.0.0.1       algashop-mongodb-3
```

O conteúdo está em `etc/hostnames/hostnames`, e o passo a passo por sistema operacional — inclusive o `sudo` do macOS/Linux e o Bloco de Notas como administrador no Windows — em `etc/hostnames/editando-arquivo-hosts.md`.

Conectando:

```bash
docker exec -it algashop-meta-algashop-mongodb-1-1 mongosh

use product_catalog
db.products.find().pretty()
db.stock_movements.find().pretty()

rs.status()      # confere o conjunto: quem é PRIMARY, quem é SECONDARY
```

Diferente do Postgres, **não há script de criação** — o Mongo cria o banco e as coleções no primeiro insert.

Quem popula as coleções é a própria aplicação: o `DataLoader` do `product-catalog` lê os JSONs de `db/testdata/` a cada inicialização.

> ⚠️ Com `algashop.data-load.auto-drop: true` (o valor atual no `application.yml`), as coleções `products` e `categories` são **apagadas e recriadas toda vez que o serviço sobe**. Alterou um documento pelo `mongosh` e reiniciou? A alteração se foi.

Os **índices** também são criados pela aplicação, a partir das anotações do agregado (`auto-index-creation`). Para conferir depois de subir:

```bash
db.products.getIndexes()
```

Devem aparecer cinco: o `_id_` automático, o `idx_product_by_brand`, o índice de texto e dois `pidx_*` compostos.

A terceira coleção, `stock_movements`, aparece na primeira entrada ou saída de estoque — e é a única que **não** é gerenciada pelo `DataLoader`: ela não está nas `sources` do YAML, então o `auto-drop` não a alcança e os movimentos acumulam entre execuções.

> Detalhes de modelagem: [`product-catalog-mongo.md`](../02-persistencia/product-catalog-mongo.md).
> Como a carga funciona: [`carga-de-dados-mongo.md`](./carga-de-dados-mongo.md).
> Índices e `explain`: [`indices-mongo.md`](../02-persistencia/indices-mongo.md).

---

## 5. Ferramentas de apoio

### WireMock (porta 8787)

Finge ser uma API externa e devolve respostas fixas, definidas em JSON:

```
etc/wiremock/
├── get-product-by-id-v1.json            # catálogo respondendo 200
├── get-product-by-id-v1-not-found.json  # catálogo respondendo 404
├── rapidex.json                         # transportadora
└── fastpay.json                         # gateway de pagamento
```

O diretório é montado como volume (`./etc/wiremock:/home/wiremock/mappings`), então **editar um JSON e reiniciar o container** já aplica a mudança — não precisa rebuildar nada.

Serve para o `ordering` rodar sem que o `product-catalog` esteja de pé. Ver [`stubs-contract-tests.md`](../03-testes-integracao/stubs-contract-tests.md).

### FastPay (porta 9995)

Gateway de pagamento simulado, fornecido como imagem pronta pela AlgaWorks (`algaworks/fastpay:latest`). Usa o banco `fastpay` no mesmo Postgres e sobe com massa de teste (`SPRING_FLYWAY_LOCATIONS` inclui `classpath:db/testdata`). É a integração externa real do `billing`.

### Stub Runner

Alternativa ao WireMock que consome os stubs gerados pelos **contract tests** do próprio `product-catalog`:

```bash
cd etc/stub-runner
./run-product-catalog-stub.sh
```

Sobe o stub do `product-catalog` na porta **8083** a partir do artefato publicado no repositório Maven local — ou seja, é preciso ter rodado `./gradlew publishToMavenLocal` no `product-catalog` antes.

---

## 6. Rodando os serviços

```bash
cd microservices/algashop-ordering
./gradlew bootRun
```

Perfis do Spring em uso:

| Perfil | Quando | Aponta para |
|---|---|---|
| `development` | padrão, rodando na sua máquina | `localhost:5433`, `localhost:8787` |
| `docker` | dentro de container | `algashop-postgres:5432`, `wiremock:8080` |
| `production` | — | configuração externa |

Trocando de perfil:

```bash
./gradlew bootRun --args='--spring.profiles.active=docker'
```

Rodando os testes:

```bash
./gradlew test              # unitários e fatias de contexto — não precisam de infra
./gradlew contractTest      # testes gerados a partir dos contratos
./gradlew integrationTest   # classes com sufixo *IT — PRECISAM do banco de pé
```

As três suítes são separadas de propósito: falha barata aparece antes de gastar tempo com a que precisa de infraestrutura.

### Bancos de teste

Cada serviço roda sua suíte contra um banco **próprio**, nunca contra o de desenvolvimento:

| Serviço | Desenvolvimento | Testes |
|---|---|---|
| `ordering`, `billing` | `ordering`, `billing` | `*_test` (PostgreSQL) |
| `product-catalog` | `product_catalog` | um Mongo **descartável**, via Testcontainers |

No `product-catalog` a separação mudou de natureza na Fase 14. Antes era um banco diferente no **mesmo** servidor; hoje os testes de integração sobem o **próprio** Mongo:

```java
// src/test/java/.../TestContainerMongoDBConfig.java
private static final MongoDBContainer mongoDBContainer =
        new MongoDBContainer("mongo:8").withReplicaSet();
```

O `withReplicaSet()` não é detalhe: sem ele o container roda como nó único e o teste de transação morre com `error 20 (IllegalOperation)`. No Testcontainers 1.x o replica set era o padrão; no 2.x virou opt-in.

`@ServiceConnection` faz o Boot ler a porta sorteada do container e sobrescrever a URI configurada — nenhum `@DynamicPropertySource` é necessário. E `./gradlew integrationTest` deixou de exigir o `docker-compose` de pé; só o Docker.

> ⚠️ **Todo `*IT` precisa importar essa configuração.** Foi exatamente assim que os três testes de concorrência ficaram órfãos: eles não a tinham, caíram na URI do `application-test-env.yml` e morreram no `Authentication failed` quando o cluster perdeu a autenticação.

O grupo de perfis abaixo continua valendo — é ele que decide contra qual banco roda um teste **sem** Testcontainers:

```yaml
# src/test/resources/application.yml — sombreia o de src/main durante os testes
spring:
  profiles:
    active: test
    group:
      test:
        - base        # o que vale em qualquer ambiente: índices, carga, nome da app
        - test-env    # só a URI, apontando para product_catalog_test
```

> ⚠️ **Não é preciosismo.** A suíte roda com `algashop.data-load.auto-drop: true`, que **apaga as coleções** antes de recarregar a massa. Apontar os testes para `product_catalog` destruiria os dados de desenvolvimento a cada `./gradlew integrationTest`. Com Testcontainers isso deixou de ser possível — mas a conferência continua valendo, e é barata:
>
> ```javascript
> use product_catalog
> db.products.countDocuments({})      // tem que estar intacto depois da suíte
> ```

---

## 7. Problemas comuns

| Sintoma | Causa provável | Solução |
|---|---|---|
| `Connection refused` na 5432 | Apontando para a porta errada | O host expõe **5433**, não 5432 |
| `database "ordering" does not exist` | O `init-user-db.sh` não rodou | `down -v` e suba de novo (apaga os dados) |
| Mongo conecta mas a coleção está sempre vazia | Propriedade errada no Boot 4 | É `spring.mongodb.uri`, não `spring.data.mongodb.uri` |
| `Authentication failed` no Mongo | URI antiga com `root:algashop` | O cluster da Fase 14 sobe **sem auth** — tire as credenciais e o `authSource` da URI |
| `Transaction numbers are only allowed on a replica set member` | Mongo rodando como nó único | Suba os três nós do compose; em teste, `withReplicaSet()` no `MongoDBContainer` |
| Driver não resolve `algashop-mongodb-2` | Arquivo `hosts` não editado | Acrescente as três linhas de `etc/hostnames/hostnames` |
| `rs.initiate` falha no segundo `up` | Conjunto já iniciado | Esperado e inofensivo — o `\|\| true` do serviço de init existe para isso |
| `ERR AUTH ... without any password configured` | Redis subiu sem senha | Falta o `.env` na raiz do meta — [detalhes](./redis.md) |
| Cache nunca popula (`DBSIZE` sempre 0) | Conexão com o Redis falhando em silêncio | O `CacheErrorHandler` engole o erro; confira o log por `Cache GET error` |
| `NotSerializableException` ao cachear | DTO sem `implements Serializable` | O serializador é o do Java; a exigência é transitiva a todos os campos |
| Anotações de cache não fazem nada | `spring.cache.type` não é `redis` | Sem ela o `RedisCacheConfig` não é registrado, e o `@EnableCaching` não acontece |
| `/actuator/health` sempre `UNKNOWN` | Indicador de service discovery registrado por dependência transitiva | `spring.cloud.discovery.client.health-indicator.enabled: false` — [detalhes](./health-checks.md) |
| `/actuator/health` devolve 200 mesmo degradado | `DEGRADED` não é mapeado para 503 | Comportamento conhecido; olhe o corpo, não o código |
| `cache` reporta `UP` com o Redis parado | Defeito conhecido no indicador | Registrado em [`health-checks.md`](./health-checks.md) |
| Serviço não acha o `product-catalog` | Nada respondendo na URL configurada | Suba o WireMock ou o Stub Runner |
| Pastas de submódulo vazias | Clone sem `--recurse-submodules` | `git submodule update --init --recursive` |
| Alterações de um serviço somem | `git submodule update` sem `--remote` | Sempre cheque o status antes |
| Dados do Mongo somem a cada restart | `data-load.auto-drop: true` | Desligue em `application.yml` — [detalhes](./carga-de-dados-mongo.md) |
| Console cheio de log do driver Mongo | `logging.level.org.mongodb.driver...: DEBUG` | Comente o bloco no `application.yml` do `product-catalog` |
| `?term=` não acha nada | Índice de texto não foi criado | Confira `spring.data.mongodb.auto-index-creation: true` e rode `db.products.getIndexes()` — [detalhes](../02-persistencia/indices-mongo.md) |
| Busca por termo não acha por marca | `$text` cobre só `name` e `description` | Comportamento atual, registrado como pendência em [`indices-mongo.md`](../02-persistencia/indices-mongo.md) |
| Listagem lenta mesmo com índice criado | Índice parcial ignorado sem `enabled: true` | Rode `.explain("executionStats")` e compare `IXSCAN`/`COLLSCAN` — [detalhes](../02-persistencia/indices-mongo.md) |
| `./gradlew test` falha por falta de banco | Teste `*IT` rodando na suíte errada | `*IT` sai em `integrationTest`; confira o sufixo da classe |
| Dados de desenvolvimento sumiram depois de rodar testes | Um `*IT` sem `TestContainerMongoDBConfig` | Todo `*IT` precisa importá-lo; sem isso ele cai na URI do `application-test-env.yml` |
| Teste de concorrência passa sempre, mesmo com código errado | As threads não chegaram a se sobrepor | O `CountDownLatch` é o que solta todas juntas — sem ele o teste não prova nada |
| Container reiniciando sem parar | Falta memória | Os limites do compose são apertados (256M–512M) |
| Container some sob carga, `Exited (137)` | OOM kill — o limite de memória do compose | `docker inspect <c> --format '{{.State.OOMKilled}}'`. O `ordering` precisou de 512M; 256M morria por volta de 1600 req/s |
| `billing-scheduler` aparece como `Exited (0)` | É o esperado | É um job, não um serviço — roda uma vez e encerra |
| Serviço para de responder e não volta, container `Up` | Fila ilimitada num `@ConcurrencyLimit` sob threads virtuais | Reinicie o container; a explicação está em [`threads-e-concorrencia.md`](./threads-e-concorrencia.md) |
| `/actuator/health` não responde (nem timeout útil) | O endpoint usa o mesmo pool de threads da aplicação | Se o pool saturou, o health também está na fila — olhe `docker stats` e o log |
| k6 roda o perfil errado sem avisar | `__ENV` não recebeu a variável | Use `-e PROFILE=volume`, e nunca o prefixo `K6_` — [detalhes](../03-testes-integracao/testes-de-carga-k6.md) |
| `Unable to load region from any of the providers in the chain` | Starter do S3 no classpath sem região configurada | Derruba o contexto **inteiro**, inclusive em teste. `spring.cloud.aws.region.static` resolve |
| Upload falha no navegador, serviço `UP` | O host da URL assinada não resolve na máquina do cliente | Acrescente as três linhas de LocalStack ao `hosts` |
| Upload falha no *preflight*, sem mensagem sobre S3 | CORS do bucket não aplicado | `awslocal s3api get-bucket-cors --bucket algashop-product-image`; o `init.sh` aplica na subida |
| LocalStack morre durante a inicialização | OOM no `s3 sync` das imagens | O limite é 1 GB e o sync roda em processo único — não paralelize |
| `component 'awsS3' is DEGRADED` | LocalStack parado | Esperado: storage é dependência **opcional**, o `readiness` continua `UP` |
| `Port 8081 was already in use` ao subir o authorization server | Porta antiga, hoje do `ordering` | Foi corrigida para **9000** na Fase 20 — confira o `application-base.yaml` |
| `invalid_client` ao pedir token | Cliente inexistente no perfil ativo | Os clientes só estão em `application-development-env.yaml`; o perfil `production` sobe sem nenhum |
| JWT válido ontem passa a dar `401` hoje | Chave de assinatura não persistida | Sem configuração, o Spring gera par novo a cada subida — reiniciar invalida os tokens |
| Tudo responde `401` depois de subir | Nenhum token sendo enviado | Os três serviços exigem token em toda rota desde a Fase 21 — só `/actuator/health/**` e o webhook do FastPay são públicos |
| `403` com token que parece certo | Escopo faltando, não autenticação | O corpo não diz qual escopo falta, de propósito; confira a matriz em [`resource-server-e-escopos.md`](../05-seguranca/resource-server-e-escopos.md) |
| `502` na compra, com o AS fora do ar | O `ordering` pede token e não consegue | Comportamento correto desde a Fase 22: o 401 deixou de virar "produto não encontrado" |
| Compra falha só no perfil `docker` | O AS não está no compose, e o issuer é nome de container | Suba o AS por `./gradlew bootRun` fora do compose — pendência registrada |
| `UnknownHostException: algashop-authorization-server` | Falta a linha no arquivo `hosts` | Está em `etc/hostnames/hostnames`; sem ela nenhum serviço resolve o issuer |
| O authorization server não sobe: banco `authorization_server` não existe | O `init-user-db.sh` só roda com o **volume vazio** | Crie o banco à mão, ou `down -v` e suba de novo (apaga tudo) |
| `FlywayValidateException` ao subir o AS | Migration já aplicada foi editada | Nunca editar `.sql` aplicado — nem para acrescentar comentário; o checksum muda |
| `/oauth2/authorize` devolve 401 no `curl` | Negociação de conteúdo | Sem `Accept: text/html` o entry point assume que quem chama é API e devolve erro OAuth2 em vez de redirecionar ao login |
| O authorization server não sobe: banco `authorization_server` não existe | O `init-user-db.sh` só roda com o **volume vazio** | Crie o banco à mão, ou `down -v` e suba de novo (apaga tudo) |
| `FlywayValidateException` ao subir o AS | Migration já aplicada foi editada | Nunca editar `.sql` aplicado — nem para acrescentar comentário; o checksum muda |
| `/oauth2/authorize` devolve 401 no `curl` | Negociação de conteúdo | Sem `Accept: text/html` o entry point assume API e devolve erro OAuth2 em vez de redirecionar ao login |

---

## Pendências registradas

Coisas quebradas ou inconsistentes na configuração, encontradas ao documentar:

- [x] ~~`algashop-ordering/src/main/resources/application.yaml` tem o bloco `spring.profiles.group` **malformado**.~~ Resolvido na Fase 15: a linha `development: development` virou `development:` com a lista abaixo, e o grupo passou a carregar `base` + `development-env` como os outros dois já faziam.
- [ ] `application-docker-env.yaml` do `ordering`: o `datasource` aponta para `algashop-postgres:5432`, mas o `flyway.url` aponta para `algashop-postgres:5433` — dentro da rede Docker a porta correta é **5432** nos dois casos.
- [x] ~~`docker-compose.services.yml` não inclui o `product-catalog`.~~ Resolvido na Fase 17: ele entrou junto com o Dockerfile que lhe faltava, esperando o `algashop-mongodb-init` com `condition: service_completed_successfully` — os nós ficam *healthy* antes de o replica set existir, então esperar por eles não bastaria.
- [x] ~~**O `billing-scheduler` continua fora do compose.**~~ Resolvido na Fase 18: entrou como job pontual, com `restart: no`, sem portas, esperando o Postgres saudável e o `fastpay` apenas iniciado (`service_started` — o fastpay não declara healthcheck).
- [ ] **O pool do Hikari nunca foi declarado** em nenhum dos serviços. O default de 10 governa o caminho de compra inteiro e não está escrito em lugar nenhum — a Fase 18 mostrou que ele é um dos quatro limitadores de valor 10 empilhados. Ver [`threads-e-concorrencia.md`](./threads-e-concorrencia.md).
- [ ] **Nenhum Dockerfile tem `HEALTHCHECK` e o compose não tem `healthcheck:` nos serviços.** Aberto desde a Fase 17 e cobrado na Fase 18: um serviço completamente travado continuou aparecendo como `Up`.

---

## Checklist — ambiente pronto

- [ ] `git submodule foreach 'git log -1 --oneline'` lista todos os seis
- [ ] `docker compose -f docker-compose.tools.yml ps` mostra tudo `healthy`
- [ ] `psql -h localhost -p 5433 -U postgres -l` lista os cinco bancos
- [ ] `rs.status()` mostra um `PRIMARY` e dois `SECONDARY` com `health: 1`
- [ ] O arquivo `hosts` tem as três entradas `algashop-mongodb-*`
- [ ] `curl localhost:8787/__admin/mappings` devolve os stubs do WireMock
- [ ] `./gradlew test` passa em cada microsserviço
