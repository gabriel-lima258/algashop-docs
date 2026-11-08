# Specification Pattern (DDD)

## O que é?

O **Specification Pattern** é um padrão de design do Domain-Driven Design (DDD) que **encapsula regras de negócio em objetos independentes, reutilizáveis e combináveis**.

Em vez de espalhar condições `if/else` complexas dentro de serviços de domínio, cada regra de negócio se torna um objeto com um único método: `isSatisfiedBy(T)`.

## O problema que resolve

Imagine uma regra de negócio: *"O cliente tem frete grátis se tiver pelo menos 200 pontos de fidelidade E pelo menos 2 compras no ano, OU se tiver 2000+ pontos."*

**Sem Specification** (abordagem procedural):

```java
public class BuyNowService {

    private final Orders orders;

    private boolean haveFreeShipping(Customer customer) {
        // Regra monolítica: difícil de ler, testar e reutilizar
        return customer.loyaltyPoints().compareTo(new LoyaltyPoints(200)) >= 0
            && orders.salesQuantityByCustomerInYear(customer.id(), Year.now()) >= 2
            || customer.loyaltyPoints().compareTo(new LoyaltyPoints(2000)) >= 0;
    }
}
```

Problemas desta abordagem:
- A regra está **acoplada** ao serviço — não pode ser reutilizada em outro contexto
- **Difícil de testar** cada sub-condição isoladamente
- **Difícil de ler** — exige interpretar precedência de `&&` e `||`
- **Viola SRP** — o serviço cuida de orquestração E de regras de negócio

---

## A interface Specification

A base do padrão é uma interface genérica com métodos de composição:

```java
public interface Specification<T> {

    // Avalia se o objeto satisfaz a regra de negócio
    boolean isSatisfiedBy(T t);

    // Composição: E lógico — ambas devem ser verdadeiras
    default Specification<T> and(Specification<T> other) {
        return t -> this.isSatisfiedBy(t) && other.isSatisfiedBy(t);
    }

    // Composição: OU lógico — pelo menos uma deve ser verdadeira
    default Specification<T> or(Specification<T> other) {
        return t -> this.isSatisfiedBy(t) || other.isSatisfiedBy(t);
    }

    // Negação — inverte o resultado da regra
    default Specification<T> not(Specification<T> other) {
        return t -> !this.isSatisfiedBy(t);
    }

    // Composição: E NÃO — this verdadeira e other falsa
    default Specification<T> andNot(Specification<T> other) {
        return t -> this.isSatisfiedBy(t) && !other.isSatisfiedBy(t);
    }
}
```

Os métodos `and()`, `or()`, `not()` e `andNot()` retornam **novas Specifications** (via lambda), permitindo compor regras complexas a partir de regras simples sem criar novas classes.

> **Localização no projeto:** `domain/model/Specification.java`

---

## Como foi aplicado no projeto AlgaShop

### 1. Specifications atômicas (regras individuais)

Cada sub-regra de negócio se tornou uma classe independente:

**`CustomerHasEnoughLoyaltyPointsSpecification`** — verifica se o cliente tem pontos suficientes:

```java
@RequiredArgsConstructor
public class CustomerHasEnoughLoyaltyPointsSpecification
        implements Specification<Customer> {

    private final LoyaltyPoints expectedLoyaltyPoints;  // limiar parametrizável

    @Override
    public boolean isSatisfiedBy(Customer customer) {
        return customer.loyaltyPoints().compareTo(expectedLoyaltyPoints) >= 0;
    }
}
```

**`CustomerHasOrderedEnoughAtYearSpecification`** — verifica se o cliente fez compras suficientes no ano:

```java
@RequiredArgsConstructor
public class CustomerHasOrderedEnoughAtYearSpecification
        implements Specification<Customer> {

    private final Orders orders;               // acesso ao repositório
    private final long expectedOrderCount;     // quantidade mínima

    @Override
    public boolean isSatisfiedBy(Customer customer) {
        return orders.salesQuantityByCustomerInYear(
                customer.id(), Year.now()
        ) >= expectedOrderCount;
    }
}
```

> Observe que cada Specification é **parametrizável** via construtor. A mesma classe `CustomerHasEnoughLoyaltyPointsSpecification` pode ser instanciada com 200 pontos (básico) ou 2000 pontos (premium), gerando comportamentos diferentes sem duplicar código.

### 2. Specification composta (combinação de regras)

A `CustomerHaveFreeShippingSpecification` **compõe** as Specifications atômicas usando os operadores `and()` e `or()`:

```java
@RequiredArgsConstructor
public class CustomerHaveFreeShippingSpecification implements Specification<Customer> {

    private final CustomerHasOrderedEnoughAtYearSpecification hasOrderedEnoughAtYearSpecification;
    private final CustomerHasEnoughLoyaltyPointsSpecification hasEnoughBasicLoyaltyPointsSpecification;
    private final CustomerHasEnoughLoyaltyPointsSpecification hasEnoughPremiumLoyaltyPointsSpecification;

    @Override
    public boolean isSatisfiedBy(Customer customer) {
        // Lê-se como a regra de negócio:
        // "pontos básicos E compras suficientes OU pontos premium"
        return hasEnoughBasicLoyaltyPointsSpecification
                .and(hasOrderedEnoughAtYearSpecification)
                .or(hasEnoughPremiumLoyaltyPointsSpecification)
                .isSatisfiedBy(customer);
    }
}
```

**Comparação visual — antes vs depois:**

| Antes (procedural) | Depois (Specification) |
|---------------------|------------------------|
| `loyaltyPoints >= 200 && salesQty >= 2 \|\| loyaltyPoints >= 2000` | `pontosBasicos.and(comprasSuficientes).or(pontosPremium)` |
| Exige interpretar `compareTo`, `&&`, `||` e precedência | Lê-se como linguagem natural |

### 3. Configuração via Spring Bean

Os limiares de negócio são definidos na configuração, não no código de domínio:

```java
@Configuration
public class SpecificationsBeansConfig {

    @Bean
    public CustomerHaveFreeShippingSpecification customerHaveFreeShippingSpecification(
            Orders orders) {
        return new CustomerHaveFreeShippingSpecification(
                orders,
                new LoyaltyPoints(200),   // pontos básicos mínimos
                2L,                        // compras mínimas no ano
                new LoyaltyPoints(2000)    // pontos premium (frete grátis direto)
        );
    }
}
```

Isso permite alterar os limiares de negócio sem modificar código de domínio — basta ajustar o Bean (ou futuramente mover para `application.properties`).

> **Localização:** `infrastructure/beans/SpecificationsBeansConfig.java`

### 4. Uso nos Domain Services

Tanto `BuyNowService` quanto `CheckoutService` reutilizam a **mesma** Specification injetada pelo Spring:

```java
@DomainService
@RequiredArgsConstructor
public class BuyNowService {

    private final CustomerHaveFreeShippingSpecification customerHaveFreeShippingSpecification;

    public Order buyNow(Product product, Customer customer, /* ... */) {
        // ... monta o pedido ...

        if (customerHaveFreeShippingSpecification.isSatisfiedBy(customer)) {
            order.changeShipping(shipping.toBuilder().cost(Money.ZERO).build());
        } else {
            order.changeShipping(shipping);
        }

        order.markAsPlaced();
        return order;
    }
}
```

A mesma Specification é reutilizada no `CheckoutService` — a regra de frete grátis é definida **uma única vez** e compartilhada entre todos os serviços que precisam dela.

---

## Diagrama da arquitetura

```
┌─────────────────────────────────────────┐
│           Domain Services               │
│  ┌──────────────┐  ┌────────────────┐   │
│  │ BuyNowService│  │CheckoutService │   │
│  └──────┬───────┘  └───────┬────────┘   │
│         │                  │             │
│         └────────┬─────────┘             │
│                  ▼                       │
│  ┌─────────────────────────────────────┐ │
│  │ CustomerHaveFreeShippingSpecification│ │  ← Specification composta
│  └──┬────────────┬─────────────────┬──┘ │
│     │  .and()    │                 │     │
│     ▼            ▼            .or()▼     │
│  ┌────────┐ ┌───────────┐ ┌────────────┐│
│  │ Basic  │ │ Ordered   │ │  Premium   ││  ← Specifications atômicas
│  │Loyalty │ │ Enough    │ │  Loyalty   ││
│  │Points  │ │ AtYear    │ │  Points    ││
│  │(≥ 200) │ │ (≥ 2)     │ │  (≥ 2000) ││
│  └────────┘ └───────────┘ └────────────┘│
└─────────────────────────────────────────┘

Regra: (BasicPoints AND OrderedEnough) OR PremiumPoints
```

---

## Vantagens do padrão

| Vantagem | Descrição |
|----------|-----------|
| **Single Responsibility** | Cada Specification encapsula uma única regra de negócio |
| **Reutilização** | A mesma Specification é usada em `BuyNowService` e `CheckoutService` |
| **Parametrização** | Uma classe, múltiplas instâncias com limiares diferentes (200 pts vs 2000 pts) |
| **Composição** | Regras complexas são montadas com `.and()`, `.or()`, `.not()` |
| **Testabilidade** | Cada sub-regra pode ser testada unitariamente de forma isolada |
| **Open/Closed** | Novas regras são adicionadas compondo Specifications existentes, sem modificar código |
| **Legibilidade** | A composição fluente se lê como linguagem natural |

---

## Quando usar

- Regras de negócio que aparecem em **múltiplos contextos** (como frete grátis no BuyNow e Checkout)
- Condições **compostas** com múltiplas sub-regras (AND, OR, NOT)
- Regras que precisam ser **testadas isoladamente**
- Limiares que podem **variar** entre ambientes ou configurações

## Quando NÃO usar

- Condições simples e pontuais (`if (order.isEmpty())`)
- Regras que aparecem em um **único lugar** e são triviais
- Validações de entrada (para isso, use Bean Validation / `@Valid`)
