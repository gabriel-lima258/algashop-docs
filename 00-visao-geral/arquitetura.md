# Arquitetura do AlgaShop

> Visão macro: quais são os serviços, por que estão separados assim, e como conversam entre si.

---

## O sistema

O AlgaShop é um e-commerce decomposto em quatro microsserviços, cada um com banco próprio e ciclo de deploy independente.

```mermaid
graph TB
    Cliente([Cliente])

    subgraph Serviços
        O["<b>ordering</b><br/>:8081<br/>pedidos e carrinho"]
        C["<b>product-catalog</b><br/>:8083<br/>produtos e categorias"]
        B["<b>billing</b><br/>:8082<br/>faturas e pagamento"]
        S["<b>billing-scheduler</b><br/>jobs agendados"]
    end

    subgraph Bancos
        PG[(PostgreSQL<br/>:5433)]
        MG[(MongoDB<br/>:27017)]
    end

    FP[FastPay<br/>gateway externo<br/>:9995]
    RX[Rapidex<br/>transportadora]

    Cliente --> O
    Cliente --> C
    O -->|HTTP: dados do produto| C
    O -->|evento: pedido confirmado| B
    B -->|HTTP| FP
    O -->|HTTP| RX
    S -->|cancela faturas expiradas| B

    O --- PG
    B --- PG
    C --- MG
```

---

## Os serviços

### `algashop-ordering` — o coração do domínio

Carrinho de compras, pedidos, clientes, checkout. É o serviço mais maduro e onde a maior parte dos conceitos de DDD foi exercitada: agregados, value objects, domain events, specifications, domain services.

| | |
|---|---|
| **Porta** | 8081 |
| **Banco** | PostgreSQL (`ordering`) |
| **Arquitetura** | Hexagonal — `core/` (domínio + aplicação + ports) e `infrastructure/` (adapters) |
| **Destaques** | DDD tático, CQRS com Criteria API, domain events, Flyway |
| **Pacote** | `com.gtech.algashop` |

```
core/
├── domain/model/        agregados, value objects, domain services
├── application/         casos de uso
└── ports/
    ├── in/              o que a aplicação OFERECE
    └── out/             o que a aplicação PRECISA
infrastructure/
├── adapters/in/web      controllers REST, listeners
├── adapters/out/        persistência JPA, notificações
└── config/
```

> [`ports-hexagonal.md`](../01-arquitetura-design/ports-hexagonal.md) · [`cqrs.md`](../01-arquitetura-design/cqrs.md) · [`specification.md`](../01-arquitetura-design/specification.md)

### `algashop-billing` — cobrança

Faturas e integração com gateway de pagamento. É o serviço que mais ensina sobre **falar com o mundo externo**: o FastPay pode estar fora do ar, pode demorar, pode recusar o cartão.

| | |
|---|---|
| **Porta** | 8082 |
| **Banco** | PostgreSQL (`billing`) |
| **Destaques** | Integração FastPay, WireMock, contract tests, Testcontainers |
| **Pacote** | `com.algaworks.algashop.billing` |

A infraestrutura de pagamento tem duas implementações — `payment/fastpay/` (real) e `payment/fake/` (para desenvolvimento e testes). Trocar uma pela outra é questão de configuração, e é a razão prática de existir uma abstração ali.

> [`stubs-contract-tests.md`](../03-testes-integracao/stubs-contract-tests.md)

### `product-catalog` — catálogo

Produtos e categorias. O único serviço em MongoDB — e a razão é o padrão de acesso: muito mais leitura que escrita, atributos que variam por tipo de produto, nenhuma transação com dinheiro.

| | |
|---|---|
| **Porta** | 8083 |
| **Banco** | MongoDB (`product_catalog`) |
| **Destaques** | Modelagem documental e desnormalização, eventos de domínio, controle de concorrência atômico, Spring Boot 4, REST Docs, contract-driven development |
| **Pacote** | `com.algaworks.algashop.product.catalog` |

> [`product-catalog-mongo.md`](../02-persistencia/product-catalog-mongo.md) · [`desnormalizacao-mongo.md`](../02-persistencia/desnormalizacao-mongo.md) · [`concorrencia-e-atomicidade.md`](../02-persistencia/concorrencia-e-atomicidade.md) · [`eventos-e-listeners.md`](../01-arquitetura-design/eventos-e-listeners.md) · [`tratamento-erros-api.md`](../03-testes-integracao/tratamento-erros-api.md)

### `billing-scheduler` — jobs agendados

O menor dos quatro. Cancela faturas expiradas periodicamente. Existe separado do `billing` de propósito: job agendado e API REST têm perfis de escala completamente diferentes — um roda de vez em quando e pode ser pesado, o outro precisa responder rápido o tempo todo.

> [`scheduled-jobs.md`](../04-infraestrutura/scheduled-jobs.md)

---

## Persistência poliglota

| Serviço | Banco | Por quê |
|---|---|---|
| `ordering` | PostgreSQL | Transações, invariantes fortes, dinheiro |
| `billing` | PostgreSQL | Idem — fatura errada é problema sério |
| `product-catalog` | MongoDB | Leitura dominante, schema flexível, volume |

Isso só é possível porque **cada serviço é dono do seu banco**. Nenhum serviço lê a tabela do outro — a comunicação é sempre pela API. Se dois serviços compartilhassem banco, mudar um schema quebraria o outro, e a independência de deploy iria embora.

> [`nosql-conceitos.md`](../02-persistencia/nosql-conceitos.md)

---

## Como os serviços conversam

### Chamada síncrona (HTTP)

`ordering` → `product-catalog` para buscar dados de produto ao montar um pedido.

O acoplamento aqui é real: se o catálogo cair, o pedido não é criado. O que reduz o risco é o **contrato explícito** — o `product-catalog` publica um contrato, e o `ordering` testa contra o stub gerado a partir dele. Nenhum dos dois precisa do outro de pé para rodar os testes.

```
product-catalog  --define-->  contrato  --gera-->  stub
                                                     ↓
                                          ordering testa contra o stub
```

### Integração externa

`billing` → FastPay (pagamento), `ordering` → Rapidex (frete). Ambas mockadas por WireMock no ambiente local.

### O que ainda não existe

Mensageria (RabbitMQ/Kafka) entre serviços. Os eventos hoje são **internos ao processo** que os publica, em dois serviços:

| Onde | Como |
|---|---|
| `ordering` | domain events publicados e consumidos via `ApplicationEventPublisher` do Spring |
| `product-catalog` | o mesmo mecanismo, por **duas** portas de saída — `ApplicationMessagePublisher` (aplicação) e `DomainEventPublisher` (domínio) —, com o consumidor da categoria rodando `@Async` |

O `product-catalog` é o caso mais interessante dos dois, porque lá o evento não é acessório: ele é o que mantém a cópia desnormalizada da categoria em dia. Um evento perdido não é só uma notificação a menos — é dado desatualizado que ninguém corrige. Ver [`eventos-e-listeners.md`](../01-arquitetura-design/eventos-e-listeners.md).

É o próximo passo natural da evolução — e o que falta não é só o broker: retentativa, dead letter, ordem e reconciliação vêm junto.

---

## Artefatos de apoio

| Onde | O quê |
|---|---|
| [`../domain-diagram/`](../domain-diagram/) | Diagramas de domínio em PDF — `Ordering.pdf`, `Product&Catalog.pdf`, `billing.pdf` |
| [`../openapi/`](../openapi/) | Especificações OpenAPI — `ordering.yml`, `billing.yml`, `product-catalog.yml` |
| `etc/wiremock/` (no meta) | Stubs das APIs externas |
| `etc/stub-runner/` (no meta) | Runner dos stubs gerados pelos contract tests |

---

## Princípios que se repetem

Padrões que aparecem em mais de um serviço — vale reconhecê-los:

1. **Domínio no centro, infraestrutura na borda.** O domínio nunca importa Spring Web, JPA ou driver de banco.
2. **Agregado protege o próprio invariante.** Validação no setter da entidade, não no controller.
3. **Leitura separada da escrita.** Comandos passam pelo agregado; consultas usam projeções diretas (CQRS).
4. **Exceção de domínio não conhece HTTP.** A tradução para status acontece só no `@RestControllerAdvice`.
5. **Integração externa sempre atrás de uma abstração.** Permite implementação fake em teste.
6. **Contrato antes da implementação.** Contract-driven development entre serviços.
