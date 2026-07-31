# AlgaShop — Portfólio de Estudos

E-commerce construído em microsserviços com **Java 21 + Spring Boot**, como projeto de estudo de arquitetura de software.

Este repositório é o caderno do projeto: cada documento registra um conceito aplicado, o problema que ele resolve, o código real onde ele aparece e as armadilhas encontradas no caminho.

---

## Os serviços

| Serviço | Porta | Banco | Destaque técnico |
|---|---|---|---|
| **`algashop-ordering`** | 8081 | PostgreSQL | DDD tático, arquitetura hexagonal, CQRS, domain events |
| **`algashop-billing`** | 8082 | PostgreSQL | Integração com gateway de pagamento, contract tests, Testcontainers |
| **`product-catalog`** | 8083 | MongoDB | Modelagem documental, Spring Boot 4, REST Docs |
| **`billing-scheduler`** | — | — | Jobs agendados, cancelamento de faturas expiradas |

> Como os serviços se conectam, quem chama quem e por quê: **[Arquitetura](./00-visao-geral/arquitetura.md)**

---

## Índice

### 00 — Visão geral

| Documento | O que você aprende |
|---|---|
| [Arquitetura](./00-visao-geral/arquitetura.md) | Mapa dos serviços, comunicação entre eles, persistência poliglota e os princípios que se repetem |
| [Linha do tempo](./00-visao-geral/linha-do-tempo.md) | A jornada em 11 fases — o que foi construído em cada etapa e por que naquela ordem |

### 01 — Arquitetura e design

| Documento | O que você aprende |
|---|---|
| [Ports & Adapters (Hexagonal)](./01-arquitetura-design/ports-hexagonal.md) | Por que separar `ports/in` de `ports/out`, quem implementa e quem consome cada porta |
| [CQS e CQRS](./01-arquitetura-design/cqrs.md) | Separar comando de consulta, as abordagens de CQRS e o que **não** confundir |
| [Specification Pattern](./01-arquitetura-design/specification.md) | Transformar regra de negócio em objeto combinável, em vez de `if` aninhado |

### 02 — Persistência

| Documento | O que você aprende |
|---|---|
| [NoSQL — conceitos](./02-persistencia/nosql-conceitos.md) | Resiliência, escalabilidade, teorema CAP e as famílias de banco NoSQL |
| [MongoDB no product-catalog](./02-persistencia/product-catalog-mongo.md) | Modelagem documental, embutir vs. referenciar, UUID como `_id`, auditoria e lock otimista |
| [Consultas dinâmicas com Criteria](./02-persistencia/consultas-mongo-criteria.md) | Filtro com N parâmetros opcionais, `$expr`, busca textual e paginação manual no Mongo |
| [Índices no MongoDB](./02-persistencia/indices-mongo.md) | `explain`, regra ESR, índice composto, índice parcial, índice de texto e ordenação por relevância |
| [Aggregation Pipeline](./02-persistencia/agregacoes-mongo.md) | O outro jeito de consultar: `$lookup` contra o N+1, campos derivados no `$project` e por que a ordem dos estágios é o custo |
| [Paginação](./02-persistencia/paginacao.md) | Paginação com Criteria API, projeção com `builder.construct()` e consulta de contagem |
| [Flyway](./02-persistencia/flyway.md) | Versionar o schema como código; por que `ddl-auto` não serve para produção |

### 03 — Testes e integração

| Documento | O que você aprende |
|---|---|
| [Stubs e contract tests](./03-testes-integracao/stubs-contract-tests.md) | Testar integração entre serviços sem subir todos: Spring Cloud Contract, WireMock, Stub Runner |
| [Tratamento de erros na API](./03-testes-integracao/tratamento-erros-api.md) | `ProblemDetail` (RFC 9457), hierarquia de exceções e quando usar 404, 422 ou 500 |

### 04 — Infraestrutura

| Documento | O que você aprende |
|---|---|
| [Ambiente local](./04-infraestrutura/ambiente-local.md) | Do clone aos serviços rodando: submódulos, Docker Compose, portas, bancos e problemas comuns |
| [Carga de dados no MongoDB](./04-infraestrutura/carga-de-dados-mongo.md) | `ApplicationRunner`, Extended JSON e por que isso **não** substitui o Flyway |
| [Docker](./04-infraestrutura/docker.md) | Build de imagem, multi-arquitetura com Buildx e publicação em registry |
| [Jobs agendados](./04-infraestrutura/scheduled-jobs.md) | `@Scheduled`, execução em ambiente distribuído e controle de concorrência |

### Artefatos

| Pasta | Conteúdo |
|---|---|
| [`domain-diagram/`](./domain-diagram/) | Diagramas de domínio em PDF — ordering, billing, product-catalog |
| [`openapi/`](./openapi/) | Especificações OpenAPI dos três serviços com API REST |

---

## Trilha de estudo sugerida

Para revisar o conteúdo do zero, nesta ordem:

**1. Entender o terreno**
[Arquitetura](./00-visao-geral/arquitetura.md) → [Linha do tempo](./00-visao-geral/linha-do-tempo.md) → [Ambiente local](./04-infraestrutura/ambiente-local.md)

**2. Como o domínio é organizado**
[Ports & Adapters](./01-arquitetura-design/ports-hexagonal.md) → [Specification](./01-arquitetura-design/specification.md) → [CQS e CQRS](./01-arquitetura-design/cqrs.md)

**3. Como os dados são guardados e consultados**
[Flyway](./02-persistencia/flyway.md) → [Paginação](./02-persistencia/paginacao.md) → [NoSQL conceitos](./02-persistencia/nosql-conceitos.md) → [MongoDB na prática](./02-persistencia/product-catalog-mongo.md) → [Consultas com Criteria](./02-persistencia/consultas-mongo-criteria.md) → [Índices](./02-persistencia/indices-mongo.md) → [Aggregation Pipeline](./02-persistencia/agregacoes-mongo.md)

**4. Como os serviços conversam e falham**
[Contract tests](./03-testes-integracao/stubs-contract-tests.md) → [Tratamento de erros](./03-testes-integracao/tratamento-erros-api.md)

**5. Como tudo roda**
[Carga de dados](./04-infraestrutura/carga-de-dados-mongo.md) → [Docker](./04-infraestrutura/docker.md) → [Jobs agendados](./04-infraestrutura/scheduled-jobs.md)

---

## Consulta rápida por assunto

| Preciso lembrar de… | Vá para |
|---|---|
| Diferença entre agregado e entidade | [Arquitetura](./00-visao-geral/arquitetura.md) |
| Onde colocar uma regra de negócio nova | [Ports & Adapters](./01-arquitetura-design/ports-hexagonal.md), [Specification](./01-arquitetura-design/specification.md) |
| Por que a listagem não devolve o objeto completo | [CQRS](./01-arquitetura-design/cqrs.md), [MongoDB](./02-persistencia/product-catalog-mongo.md) |
| Embutir ou referenciar no Mongo | [MongoDB](./02-persistencia/product-catalog-mongo.md) |
| Montar um filtro com N parâmetros opcionais | [Consultas com Criteria](./02-persistencia/consultas-mongo-criteria.md) |
| Comparar dois campos do mesmo documento | [Consultas com Criteria](./02-persistencia/consultas-mongo-criteria.md) |
| Juntar duas coleções no Mongo | [Aggregation Pipeline](./02-persistencia/agregacoes-mongo.md) |
| Resolver N+1 no Mongo | [Aggregation Pipeline](./02-persistencia/agregacoes-mongo.md) |
| Calcular campo derivado no banco em vez de em Java | [Aggregation Pipeline](./02-persistencia/agregacoes-mongo.md) |
| Saber se minha consulta usa índice | [Índices no MongoDB](./02-persistencia/indices-mongo.md) |
| Por que meu índice existe e não foi usado | [Índices no MongoDB](./02-persistencia/indices-mongo.md) |
| Ordenar resultado de busca por relevância | [Índices no MongoDB](./02-persistencia/indices-mongo.md) |
| Popular o Mongo com dados de teste | [Carga de dados](./04-infraestrutura/carga-de-dados-mongo.md) |
| Devolver 404 ou 422 | [Tratamento de erros](./03-testes-integracao/tratamento-erros-api.md) |
| Criar uma migration | [Flyway](./02-persistencia/flyway.md) |
| Testar sem subir o outro serviço | [Contract tests](./03-testes-integracao/stubs-contract-tests.md) |
| Separar teste rápido de teste que precisa de banco | [Contract tests](./03-testes-integracao/stubs-contract-tests.md) |
| Em que porta roda cada coisa | [Ambiente local](./04-infraestrutura/ambiente-local.md) |
| Comandos de submódulo Git | [Ambiente local](./04-infraestrutura/ambiente-local.md) |

---

## Como começar a rodar

```bash
git clone --recurse-submodules https://github.com/gabriel-lima258/algashop-meta.git
cd algashop-meta

docker compose -f docker-compose.tools.yml up -d

cd microservices/algashop-ordering
./gradlew bootRun
```

Passo a passo completo, mapa de portas e solução de problemas: **[Ambiente local](./04-infraestrutura/ambiente-local.md)**.

---

## Stack

**Java 21** · **Spring Boot 3.5** (`ordering`, `billing`, `billing-scheduler`) e **4.0** (`product-catalog`) · Spring Data JPA · Spring Data MongoDB · PostgreSQL 17 · MongoDB 8 · Flyway · Gradle · Spring Cloud Contract · WireMock · Testcontainers · JUnit 5 · AssertJ · ModelMapper · Lombok · Docker Compose

---

## Sobre a organização deste repositório

As pastas são numeradas para dar **ordem de leitura**, não só agrupamento — a numeração vai do mais conceitual (visão geral, design) ao mais operacional (infraestrutura).

Cada documento segue a mesma estrutura:

1. **O problema** que o conceito resolve
2. **A solução** explicada com código real do projeto
3. **Armadilhas** encontradas na prática
4. **Pendências registradas** — o que ficou por fazer, declarado abertamente
5. **Checklist** de revisão

O item 4 é proposital: documentação que só mostra o que deu certo não ajuda a estudar. As pendências marcam onde o próximo aprendizado começa.
