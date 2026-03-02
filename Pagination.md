# Paginação com JPA Criteria API e Spring Data

## Índice

1. [Visão Geral](#visão-geral)
2. [Por que usar Criteria API para paginação?](#por-que-usar-criteria-api-para-paginação)
3. [Arquitetura da Solução](#arquitetura-da-solução)
4. [Os DTOs de Projeção](#os-dtos-de-projeção)
5. [Fluxo Completo: Método `filter()`](#fluxo-completo-método-filter)
6. [Consulta de Contagem: `countTotalQueryResults()`](#consulta-de-contagem-counttotalqueryresults)
7. [Consulta de Dados: `filterQuery()`](#consulta-de-dados-filterquery)
8. [Projeção com `builder.construct()` — O Coração da Implementação](#projeção-com-builderconstruct--o-coração-da-implementação)
9. [Paginação com `setFirstResult` e `setMaxResults`](#paginação-com-setfirstresult-e-setmaxresults)
10. [Montagem do `PageImpl`](#montagem-do-pageimpl)
11. [Diferença entre `findById()` e `filter()`](#diferença-entre-findbyid-e-filter)
12. [O SQL Gerado](#o-sql-gerado)
13. [Diagrama de Sequência](#diagrama-de-sequência)
14. [Glossário](#glossário)

---

## Visão Geral

A classe `OrderQueryServiceImpl` implementa paginação manual usando a **JPA Criteria API** em conjunto com o **Spring Data `Page`**. O objetivo é listar pedidos de forma paginada, retornando apenas os campos necessários para a tela de listagem — sem carregar entidades completas com seus relacionamentos pesados (itens, endereços de cobrança, dados de envio).

A implementação segue o padrão **Two-Query Pagination** (Paginação com Duas Consultas):

1. **Primeira consulta**: conta o total de registros (`SELECT COUNT(*)`)
2. **Segunda consulta**: busca apenas a página solicitada com projeção direta em DTOs

```
Cliente solicita página 2 (size=15)
        │
        ▼
┌─────────────────────────┐
│  COUNT(*) = 87 registros │  ← 1ª consulta
└─────────────────────────┘
        │
        ▼
┌─────────────────────────┐
│  SELECT ... LIMIT 15     │  ← 2ª consulta (registros 30-44)
│  OFFSET 30               │
└─────────────────────────┘
        │
        ▼
┌─────────────────────────┐
│  PageImpl(dados, pag, 87)│  ← Resultado encapsulado
└─────────────────────────┘
```

---

## Por que usar Criteria API para paginação?

Existem várias formas de implementar paginação com Spring/JPA. Cada uma tem trade-offs:

| Abordagem | Prós | Contras |
|-----------|------|---------|
| **Spring Data `findAll(Pageable)`** | Simples, zero código manual | Retorna entidades completas; sem controle fino da projeção |
| **JPQL com `@Query`** | Bom para consultas estáticas | Pouco flexível para filtros dinâmicos |
| **Criteria API (esta implementação)** | Projeção precisa em DTOs, filtros dinâmicos, type-safe | Mais verboso, curva de aprendizado maior |
| **QueryDSL** | Fluente, type-safe, conciso | Dependência extra, geração de código |

A **Criteria API** foi escolhida aqui porque:

- **Projeção direta em DTO**: o `builder.construct()` faz o JPA instanciar o DTO diretamente no `SELECT`, sem passar pela entidade
- **Filtros dinâmicos**: a estrutura permite adicionar `WHERE` condicionais facilmente (ex: filtrar por status, data, cliente)
- **Controle total**: offset, limit, joins, e a contagem são definidos programaticamente
- **Sem dependências extras**: a Criteria API faz parte da especificação JPA (Jakarta Persistence)

---

## Arquitetura da Solução

```
┌──────────────────────────────────────────────────────────┐
│                    Application Layer                      │
│                                                          │
│  OrderQueryService (interface)                           │
│  ├── findById(String): OrderDetailOutput                 │
│  └── filter(PageFilter): Page<OrderSummaryOutput>        │
│                                                          │
│  PageFilter (DTO de entrada: page + size)                │
│  OrderSummaryOutput (DTO de saída: dados resumidos)      │
│  OrderDetailOutput (DTO de saída: dados completos)       │
│  CustomerMinimalOutput (DTO aninhado: dados do cliente)  │
└──────────────────────────────────────────────────────────┘
                          │
                          │ implementa
                          ▼
┌──────────────────────────────────────────────────────────┐
│                  Infrastructure Layer                     │
│                                                          │
│  OrderQueryServiceImpl                                   │
│  ├── usa EntityManager + Criteria API para filter()      │
│  └── usa OrderJpaEntityRepository + Mapper para findById │
│                                                          │
│  OrderPersistenceEntity (entidade JPA mapeada)           │
│  CustomerPersistenceEntity (entidade JPA relacionada)    │
└──────────────────────────────────────────────────────────┘
```

Observe a separação CQRS:
- A **interface** `OrderQueryService` fica na camada de aplicação (não conhece JPA)
- A **implementação** `OrderQueryServiceImpl` fica na camada de infraestrutura (conhece JPA, Criteria API, entidades de persistência)
- Os **DTOs** (`OrderSummaryOutput`, `PageFilter`) ficam na camada de aplicação — são o contrato de comunicação

---

## Os DTOs de Projeção

### PageFilter — Entrada da Paginação

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class PageFilter {
    private int size = 15;   // quantidade de itens por página
    private int page = 0;    // número da página (zero-based)
}
```

É intencionalmente simples. Ele carrega apenas dois parâmetros: **qual página** e **quantos itens por página**. Os valores padrão (`page=0`, `size=15`) garantem que, mesmo sem parâmetros, a primeira página com 15 resultados é retornada.

### OrderSummaryOutput — Saída da Listagem

```java
@Data
@AllArgsConstructor    // ← ESSENCIAL para builder.construct() funcionar
@NoArgsConstructor
@Builder
public class OrderSummaryOutput {
    private String id;
    private CustomerMinimalOutput customer;   // DTO aninhado
    private Integer totalItems;
    private BigDecimal totalAmount;
    private OffsetDateTime placedAt;
    private OffsetDateTime paidAt;
    private OffsetDateTime readyAt;
    private OffsetDateTime canceledAt;
    private String status;
    private String paymentMethod;
}
```

**Por que `@AllArgsConstructor` é obrigatório?**

O `builder.construct()` da Criteria API funciona assim: ele gera um SQL que retorna colunas na ordem exata dos argumentos passados, e depois chama o **construtor com todos os argumentos** da classe para instanciar o DTO. Se não existir um construtor que aceite todos os campos na ordem correta, o JPA lança uma exceção em runtime.

**O que NÃO está aqui?**

Comparado com `OrderDetailOutput` (usado no `findById`), o `OrderSummaryOutput` **omite deliberadamente**:
- `List<OrderItemOutput> items` — os itens do pedido
- `ShippingData shipping` — dados de envio
- `BillingData billing` — dados de cobrança

Isso significa que a consulta de listagem **não faz JOIN** com a tabela de itens, nem carrega as colunas de billing/shipping. Resultado: consulta mais rápida e menos dados transferidos.

### CustomerMinimalOutput — DTO Aninhado

```java
@Data
@AllArgsConstructor    // ← Também essencial para o construct aninhado
@NoArgsConstructor
@Builder
public class CustomerMinimalOutput {
    private UUID id;
    private String firstName;
    private String lastName;
    private String email;
    private String document;
    private String phone;
}
```

Este DTO é construído **dentro** do `SELECT` pelo Criteria API, através de um `builder.construct()` aninhado. O JPA navega o relacionamento `@ManyToOne` da entidade `OrderPersistenceEntity → CustomerPersistenceEntity` e extrai os campos diretamente, sem carregar a entidade `Customer` completa no contexto de persistência.

---

## Fluxo Completo: Método `filter()`

```java
@Override
public Page<OrderSummaryOutput> filter(PageFilter filter) {
    // PASSO 1: Conta o total de registros
    Long totalQueryResults = countTotalQueryResults(filter);

    // PASSO 2: Se não há resultados, retorna página vazia (evita consulta desnecessária)
    if (totalQueryResults.equals(0L)) {
        PageRequest pageRequest = PageRequest.of(filter.getPage(), filter.getSize());
        return new PageImpl<>(new ArrayList<>(), pageRequest, totalQueryResults);
    }

    // PASSO 3: Busca os dados da página solicitada
    return filterQuery(filter, totalQueryResults);
}
```

### Por que contar primeiro?

A contagem prévia serve a dois propósitos:

1. **Otimização**: se não há registros, evita executar a consulta principal (que é mais pesada por envolver JOIN e projeção)
2. **Informação de paginação**: o `PageImpl` do Spring Data precisa saber o **total de registros** para calcular:
   - `totalPages` (total de páginas)
   - `hasNext()` (existe próxima página?)
   - `hasPrevious()` (existe página anterior?)
   - `isFirst()` / `isLast()` (é a primeira/última página?)

Sem o total, a API não conseguiria informar ao frontend quantas páginas existem.

---

## Consulta de Contagem: `countTotalQueryResults()`

```java
private Long countTotalQueryResults(PageFilter filter) {
    // 1. Obtém o CriteriaBuilder — fábrica de consultas
    CriteriaBuilder builder = entityManager.getCriteriaBuilder();

    // 2. Cria uma CriteriaQuery que retorna Long (o resultado do COUNT)
    CriteriaQuery<Long> criteriaQuery = builder.createQuery(Long.class);

    // 3. Define a tabela raiz (FROM order)
    Root<OrderPersistenceEntity> root = criteriaQuery.from(OrderPersistenceEntity.class);

    // 4. Cria a expressão COUNT(root) — equivale a COUNT(*)
    Expression<Long> count = builder.count(root);

    // 5. Define o SELECT: SELECT COUNT(*) FROM order
    criteriaQuery.select(count);

    // 6. Executa e retorna o resultado único
    TypedQuery<Long> query = entityManager.createQuery(criteriaQuery);
    return query.getSingleResult();
}
```

**SQL gerado (equivalente):**
```sql
SELECT COUNT(*) FROM "order"
```

### Anatomia da Criteria API aqui

| Componente | Papel | Equivalente SQL |
|-----------|-------|-----------------|
| `CriteriaBuilder` | Fábrica que cria queries, expressões e predicados | — |
| `CriteriaQuery<Long>` | A query em si, tipada para retornar `Long` | `SELECT ... : Long` |
| `Root<OrderPersistenceEntity>` | A tabela raiz (FROM) | `FROM "order"` |
| `builder.count(root)` | Expressão de agregação | `COUNT(*)` |
| `criteriaQuery.select(count)` | Define o que será selecionado | `SELECT COUNT(*)` |
| `entityManager.createQuery(criteriaQuery)` | Compila para JPQL/SQL | Prepara a query |
| `query.getSingleResult()` | Executa e retorna o valor único | Executa o SQL |

---

## Consulta de Dados: `filterQuery()`

Esta é a parte mais complexa e interessante da implementação:

```java
private Page<OrderSummaryOutput> filterQuery(PageFilter filter, Long totalQueryResults) {
    // 1. Obtém o CriteriaBuilder
    CriteriaBuilder builder = entityManager.getCriteriaBuilder();

    // 2. Cria uma query tipada para OrderSummaryOutput (não para a entidade!)
    CriteriaQuery<OrderSummaryOutput> criteriaQuery = builder.createQuery(OrderSummaryOutput.class);

    // 3. Define a tabela raiz
    Root<OrderPersistenceEntity> root = criteriaQuery.from(OrderPersistenceEntity.class);

    // 4. Navega para o relacionamento com Customer
    Path<Object> customer = root.get("customer");

    // 5. Define o SELECT com projeção em DTOs (explicado em detalhe abaixo)
    criteriaQuery.select(
            builder.construct(OrderSummaryOutput.class,
                    root.get("id"),
                    builder.construct(CustomerMinimalOutput.class,
                            customer.get("id"),
                            customer.get("firstName"),
                            customer.get("lastName"),
                            customer.get("email"),
                            customer.get("document"),
                            customer.get("phone")
                    ),
                    root.get("totalItems"),
                    root.get("totalAmount"),
                    root.get("placedAt"),
                    root.get("paidAt"),
                    root.get("readyAt"),
                    root.get("canceledAt"),
                    root.get("status"),
                    root.get("paymentMethod")
            )
    );

    // 6. Compila a query
    TypedQuery<OrderSummaryOutput> typedQuery = entityManager.createQuery(criteriaQuery);

    // 7. Aplica paginação: OFFSET e LIMIT
    typedQuery.setFirstResult(filter.getSize() * filter.getPage());  // OFFSET
    typedQuery.setMaxResults(filter.getSize());                       // LIMIT

    // 8. Monta o PageImpl com resultados + metadados de paginação
    PageRequest pageRequest = PageRequest.of(filter.getPage(), filter.getSize());
    return new PageImpl<>(typedQuery.getResultList(), pageRequest, totalQueryResults);
}
```

---

## Projeção com `builder.construct()` — O Coração da Implementação

O `builder.construct()` é o que diferencia esta implementação de um simples `findAll()`. Ele instrui o JPA/Hibernate a:

1. Gerar um `SELECT` apenas com as colunas listadas (não todas as colunas da tabela)
2. Instanciar o DTO diretamente a partir do resultado da query, **sem criar a entidade JPA intermediária**

### Construção Aninhada

```java
builder.construct(OrderSummaryOutput.class,        // DTO externo
    root.get("id"),                                 // campo 1: order.id
    builder.construct(CustomerMinimalOutput.class,  // campo 2: DTO aninhado
        customer.get("id"),                         //   subcampo 1: customer.id
        customer.get("firstName"),                  //   subcampo 2: customer.first_name
        customer.get("lastName"),                   //   subcampo 3: customer.last_name
        customer.get("email"),                      //   subcampo 4: customer.email
        customer.get("document"),                   //   subcampo 5: customer.document
        customer.get("phone")                       //   subcampo 6: customer.phone
    ),
    root.get("totalItems"),                         // campo 3: order.total_items
    root.get("totalAmount"),                        // campo 4: order.total_amount
    root.get("placedAt"),                           // campo 5: order.placed_at
    root.get("paidAt"),                             // campo 6: order.paid_at
    root.get("readyAt"),                            // campo 7: order.ready_at
    root.get("canceledAt"),                         // campo 8: order.canceled_at
    root.get("status"),                             // campo 9: order.status
    root.get("paymentMethod")                       // campo 10: order.payment_method
)
```

### Como funciona internamente

```
                    builder.construct(OrderSummaryOutput.class, ...)
                                        │
                    ┌───────────────────┼───────────────────────┐
                    │                   │                       │
              root.get("id")    builder.construct(...)    root.get("totalItems")
                    │           CustomerMinimalOutput           │
                    │                   │                       │
                    ▼                   ▼                       ▼
              ┌──────────┐    ┌──────────────────┐     ┌──────────────┐
              │ order.id │    │ customer.id      │     │ total_items  │
              │  (Long)  │    │ customer.f_name  │     │  (Integer)   │
              └──────────┘    │ customer.l_name  │     └──────────────┘
                              │ customer.email   │             ...
                              │ customer.document│
                              │ customer.phone   │
                              └──────────────────┘
                                        │
                                        ▼
                              new CustomerMinimalOutput(
                                  id, firstName, lastName,
                                  email, document, phone
                              )
                                        │
                                        ▼
                    new OrderSummaryOutput(
                        id,
                        customerMinimalOutput,  ← objeto já construído
                        totalItems, totalAmount,
                        placedAt, paidAt, readyAt, canceledAt,
                        status, paymentMethod
                    )
```

**Ponto crucial**: a ordem dos argumentos no `builder.construct()` **deve corresponder exatamente** à ordem dos parâmetros no construtor do DTO (`@AllArgsConstructor`). Se a ordem estiver errada, os valores serão atribuídos aos campos errados ou uma exceção será lançada por incompatibilidade de tipos.

---

## Paginação com `setFirstResult` e `setMaxResults`

```java
typedQuery.setFirstResult(filter.getSize() * filter.getPage());  // OFFSET
typedQuery.setMaxResults(filter.getSize());                       // LIMIT
```

Esses dois métodos da JPA controlam a **janela de resultados** retornada:

| Página | size | `setFirstResult` (OFFSET) | `setMaxResults` (LIMIT) | Registros retornados |
|--------|------|---------------------------|-------------------------|---------------------|
| 0      | 15   | `15 × 0 = 0`             | 15                      | 1° ao 15°           |
| 1      | 15   | `15 × 1 = 15`            | 15                      | 16° ao 30°          |
| 2      | 15   | `15 × 2 = 30`            | 15                      | 31° ao 45°          |
| 3      | 10   | `10 × 3 = 30`            | 10                      | 31° ao 40°          |

O Hibernate traduz isso para cláusulas SQL nativas:
- **PostgreSQL**: `LIMIT 15 OFFSET 30`
- **MySQL**: `LIMIT 30, 15`
- **Oracle**: usa `ROWNUM` ou `FETCH FIRST`
- **H2 (testes)**: `LIMIT 15 OFFSET 30`

---

## Montagem do `PageImpl`

```java
PageRequest pageRequest = PageRequest.of(filter.getPage(), filter.getSize());
return new PageImpl<>(typedQuery.getResultList(), pageRequest, totalQueryResults);
```

O `PageImpl` é a implementação concreta da interface `Page` do Spring Data. Ele recebe três parâmetros:

| Parâmetro | Tipo | Descrição |
|-----------|------|-----------|
| `content` | `List<OrderSummaryOutput>` | Os dados da página atual |
| `pageable` | `Pageable` | Metadados da requisição (página, tamanho) |
| `total` | `long` | Total de registros no banco (todas as páginas) |

A partir disso, o Spring calcula automaticamente:

```json
{
  "content": [ ... ],          // ← os DTOs da página
  "pageable": {
    "pageNumber": 2,           // ← página solicitada
    "pageSize": 15             // ← tamanho da página
  },
  "totalElements": 87,         // ← total de registros
  "totalPages": 6,             // ← ceil(87 / 15) = 6
  "first": false,              // ← não é a primeira página
  "last": false,               // ← não é a última página
  "numberOfElements": 15,      // ← itens nesta página
  "empty": false               // ← página não está vazia
}
```

---

## Diferença entre `findById()` e `filter()`

Os dois métodos da `OrderQueryServiceImpl` usam estratégias **completamente diferentes**:

| Aspecto | `findById()` | `filter()` |
|---------|-------------|------------|
| **Retorno** | `OrderDetailOutput` (completo) | `Page<OrderSummaryOutput>` (resumido) |
| **Estratégia** | Carrega entidade JPA + converte com Mapper | Projeção direta via Criteria API |
| **Dados do cliente** | Completos (via entidade) | Mínimos (`CustomerMinimalOutput`) |
| **Itens do pedido** | Sim (`List<OrderItemOutput>`) | Não |
| **Billing/Shipping** | Sim | Não |
| **Usa EntityManager** | Não (usa Repository) | Sim (Criteria API) |
| **Usa Mapper** | Sim | Não |
| **Nº de queries SQL** | 1 (+ lazy loads possíveis) | 2 (count + data) |

```
findById("abc-123")                    filter(page=0, size=15)
       │                                        │
       ▼                                        ▼
  Repository.findById()               countTotalQueryResults()
       │                                        │
       ▼                                        ▼
  OrderPersistenceEntity              Se total > 0: filterQuery()
  (entidade completa com                        │
   items, billing, shipping)                    ▼
       │                              OrderSummaryOutput (DTO leve)
       ▼                              direto do SELECT, sem entidade
  Mapper.convert(entity, DTO)
       │
       ▼
  OrderDetailOutput
  (DTO completo)
```

---

## O SQL Gerado

### Consulta de contagem

```sql
SELECT COUNT(o1_0.id)
FROM "order" o1_0
```

### Consulta de dados (página 0, size 15)

```sql
SELECT o1_0.id,
       c1_0.id,
       c1_0.first_name,
       c1_0.last_name,
       c1_0.email,
       c1_0.document,
       c1_0.phone,
       o1_0.total_items,
       o1_0.total_amount,
       o1_0.placed_at,
       o1_0.paid_at,
       o1_0.ready_at,
       o1_0.canceled_at,
       o1_0.status,
       o1_0.payment_method
FROM "order" o1_0
JOIN customer c1_0 ON c1_0.id = o1_0.customer_id
LIMIT 15 OFFSET 0
```

Note que:
- **Apenas 15 colunas** são selecionadas (não todas as 30+ colunas da tabela order)
- O **JOIN com customer** é automático (o JPA sabe que `root.get("customer")` requer um JOIN por causa do `@ManyToOne`)
- **Não há JOIN com order_item** — a tabela de itens nem é tocada
- As colunas de billing e shipping **não aparecem** no SELECT

---

## Diagrama de Sequência

```
Controller          OrderQueryServiceImpl       EntityManager         Database
    │                       │                       │                    │
    │  filter(page=2,       │                       │                    │
    │         size=15)      │                       │                    │
    │──────────────────────>│                       │                    │
    │                       │                       │                    │
    │                       │  countTotalQuery()    │                    │
    │                       │──────────────────────>│                    │
    │                       │                       │  SELECT COUNT(*)   │
    │                       │                       │───────────────────>│
    │                       │                       │        87          │
    │                       │                       │<───────────────────│
    │                       │         87            │                    │
    │                       │<──────────────────────│                    │
    │                       │                       │                    │
    │                       │  87 > 0, então        │                    │
    │                       │  filterQuery()        │                    │
    │                       │──────────────────────>│                    │
    │                       │                       │  SELECT ... JOIN   │
    │                       │                       │  LIMIT 15 OFFSET 30│
    │                       │                       │───────────────────>│
    │                       │                       │  15 registros      │
    │                       │                       │<───────────────────│
    │                       │  List<DTO>            │                    │
    │                       │<──────────────────────│                    │
    │                       │                       │                    │
    │  PageImpl(            │                       │                    │
    │    content=15 DTOs,   │                       │                    │
    │    page=2, size=15,   │                       │                    │
    │    total=87           │                       │                    │
    │  )                    │                       │                    │
    │<──────────────────────│                       │                    │
```

---

## Glossário

| Termo | Definição |
|-------|-----------|
| **Criteria API** | API programática da especificação JPA para construir consultas de forma type-safe, sem escrever JPQL/SQL como strings |
| **CriteriaBuilder** | Fábrica central da Criteria API. Cria queries, expressões (`count`, `sum`), predicados (`equal`, `greaterThan`) e construções de DTOs (`construct`) |
| **CriteriaQuery** | Representa a consulta em si. É tipada pelo tipo de retorno (`CriteriaQuery<Long>` para count, `CriteriaQuery<OrderSummaryOutput>` para dados) |
| **Root** | Representa a entidade raiz da consulta (o `FROM`). A partir dele, navegamos campos (`root.get("status")`) e relacionamentos (`root.get("customer")`) |
| **Path** | Representa a navegação até um campo ou relacionamento. `root.get("customer")` retorna um `Path` que pode ser navegado mais fundo: `customer.get("firstName")` |
| **builder.construct()** | Instrução que diz ao JPA: "instancie esta classe usando estes campos do SELECT como argumentos do construtor". Permite projeção direta em DTOs sem passar pela entidade |
| **TypedQuery** | A query compilada, pronta para ser parametrizada e executada. Suporta `setFirstResult` (OFFSET) e `setMaxResults` (LIMIT) |
| **PageImpl** | Implementação concreta de `Page` do Spring Data. Encapsula a lista de resultados + metadados de paginação (total de elementos, número da página, tamanho) |
| **PageRequest** | Implementação de `Pageable`. Encapsula a solicitação de paginação (qual página, qual tamanho, qual ordenação) |
| **PageFilter** | DTO customizado da aplicação que transporta os parâmetros de paginação do controller até o service |
| **Projeção** | Técnica de selecionar apenas os campos necessários no SELECT, em vez de carregar a entidade completa. Reduz tráfego de rede e uso de memória |
| **Two-Query Pagination** | Padrão onde a paginação é feita em duas etapas: primeiro conta-se o total (para metadados), depois busca-se a página de dados |
