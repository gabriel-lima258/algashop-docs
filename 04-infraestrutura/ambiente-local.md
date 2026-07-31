# Ambiente local

> Do `git clone` até os serviços rodando. Portas, bancos, ferramentas de apoio e os problemas mais comuns.

---

## 1. Clonar o projeto (com submódulos)

O `algashop-meta` **não contém código** — ele agrega cinco repositórios independentes como submódulos Git:

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

### Cenário B — tudo em container

```bash
docker compose -f docker-compose.services.yml up -d
```

Isso exige que as imagens já existam localmente (`algashop/ordering:dev` e `algashop/billing:dev`) — ver [`docker.md`](./docker.md) para gerá-las.

### Comandos úteis

```bash
docker compose -f docker-compose.tools.yml ps         # o que está de pé
docker compose -f docker-compose.tools.yml logs -f algashop-mongodb
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
| MongoDB | **27017** | 27017 | |
| WireMock | **8787** | 8080 | mock de APIs externas |
| FastPay | **9995** | 9995 | gateway de pagamento simulado |

> ⚠️ **A porta 5433 não é engano.** O Postgres do projeto é exposto em `5433` no host justamente para não conflitar com uma instalação nativa de PostgreSQL, que ocupa a `5432`. Dentro da rede Docker os containers continuam falando na `5432`.
>
> Por isso as URLs mudam conforme de onde você conecta:
> - Da sua máquina (perfil `development`): `jdbc:postgresql://localhost:5433/ordering`
> - De dentro de um container (perfil `docker`): `jdbc:postgresql://algashop-postgres:5432/ordering`

O mesmo raciocínio vale para o WireMock: `http://localhost:8787` de fora, `http://wiremock:8080` de dentro.

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

```yaml
algashop-mongodb:
  image: mongo:8
  ports:
    - 27017:27017
  environment:
    MONGO_INITDB_ROOT_USERNAME: root
    MONGO_INITDB_ROOT_PASSWORD: algashop
  healthcheck:
    test: [ "CMD", "mongosh", "--eval", "db.adminCommand('ping')" ]
```

Conectando:

```bash
docker exec -it $(docker ps -qf name=mongodb) mongosh -u root -p algashop --authenticationDatabase admin

use product_catalog
db.products.find().pretty()
db.categories.find().pretty()
```

O `--authenticationDatabase admin` (equivalente ao `?authSource=admin` da URI) é obrigatório: o usuário `root` foi criado no banco `admin`, mas os dados vivem em `product_catalog`. Sem isso, o Mongo procura as credenciais no banco errado e recusa a conexão.

Diferente do Postgres, **não há script de criação** — o Mongo cria o banco e as coleções no primeiro insert.

Quem popula as coleções é a própria aplicação: o `DataLoader` do `product-catalog` lê os JSONs de `db/testdata/` a cada inicialização.

> ⚠️ Com `algashop.data-load.auto-drop: true` (o valor atual no `application.yml`), as coleções `products` e `categories` são **apagadas e recriadas toda vez que o serviço sobe**. Alterou um documento pelo `mongosh` e reiniciou? A alteração se foi.

Os **índices** também são criados pela aplicação, a partir das anotações do agregado (`auto-index-creation`). Para conferir depois de subir:

```bash
db.products.getIndexes()
```

Devem aparecer cinco: o `_id_` automático, o `idx_product_by_brand`, o índice de texto e dois `pidx_*` compostos.

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

Rodando os testes (exigem o Postgres de pé — usam os bancos `*_test`):

```bash
./gradlew test
```

---

## 7. Problemas comuns

| Sintoma | Causa provável | Solução |
|---|---|---|
| `Connection refused` na 5432 | Apontando para a porta errada | O host expõe **5433**, não 5432 |
| `database "ordering" does not exist` | O `init-user-db.sh` não rodou | `down -v` e suba de novo (apaga os dados) |
| Mongo conecta mas a coleção está sempre vazia | Propriedade errada no Boot 4 | É `spring.mongodb.uri`, não `spring.data.mongodb.uri` |
| `Authentication failed` no Mongo | Falta o `authSource` | Acrescente `?authSource=admin` na URI |
| Serviço não acha o `product-catalog` | Nada respondendo na URL configurada | Suba o WireMock ou o Stub Runner |
| Pastas de submódulo vazias | Clone sem `--recurse-submodules` | `git submodule update --init --recursive` |
| Alterações de um serviço somem | `git submodule update` sem `--remote` | Sempre cheque o status antes |
| Dados do Mongo somem a cada restart | `data-load.auto-drop: true` | Desligue em `application.yml` — [detalhes](./carga-de-dados-mongo.md) |
| Console cheio de log do driver Mongo | `logging.level.org.mongodb.driver...: DEBUG` | Comente o bloco no `application.yml` do `product-catalog` |
| `?term=` não acha nada | Índice de texto não foi criado | Confira `spring.data.mongodb.auto-index-creation: true` e rode `db.products.getIndexes()` — [detalhes](../02-persistencia/indices-mongo.md) |
| Busca por termo não acha por marca | `$text` cobre só `name` e `description` | Comportamento atual, registrado como pendência em [`indices-mongo.md`](../02-persistencia/indices-mongo.md) |
| Listagem lenta mesmo com índice criado | Índice parcial ignorado sem `enabled: true` | Rode `.explain("executionStats")` e compare `IXSCAN`/`COLLSCAN` — [detalhes](../02-persistencia/indices-mongo.md) |
| `./gradlew test` falha por falta de banco | Teste `*IT` rodando na suíte errada | `*IT` sai em `integrationTest`; confira o sufixo da classe |
| Container reiniciando sem parar | Falta memória | Os limites do compose são apertados (256M–512M) |

---

## Pendências registradas

Coisas quebradas ou inconsistentes na configuração, encontradas ao documentar:

- [ ] `algashop-ordering/src/main/resources/application.yaml` tem o bloco `spring.profiles.group` **malformado** — a linha `development: development` seguida de uma lista indentada não é YAML válido. Os grupos de perfil provavelmente não estão sendo aplicados como o esperado.
- [ ] `application-docker-env.yaml` do `ordering`: o `datasource` aponta para `algashop-postgres:5432`, mas o `flyway.url` aponta para `algashop-postgres:5433` — dentro da rede Docker a porta correta é **5432** nos dois casos.
- [ ] `docker-compose.services.yml` não inclui o `product-catalog` nem o `billing-scheduler` — só `ordering` e `billing`.
- [ ] O `product-catalog` não tem serviço no compose, então o Cenário B não cobre o catálogo.

---

## Checklist — ambiente pronto

- [ ] `git submodule foreach 'git log -1 --oneline'` lista todos os cinco
- [ ] `docker compose -f docker-compose.tools.yml ps` mostra tudo `healthy`
- [ ] `psql -h localhost -p 5433 -U postgres -l` lista os cinco bancos
- [ ] `mongosh` conecta com `--authenticationDatabase admin`
- [ ] `curl localhost:8787/__admin/mappings` devolve os stubs do WireMock
- [ ] `./gradlew test` passa em cada microsserviço
