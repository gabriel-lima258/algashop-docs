# Ports & Adapters — Portas de Entrada (in) e Saída (out)

Este documento explica de forma didática o conceito de **Ports** (portas) na Arquitetura Hexagonal, como ele é aplicado no projeto `algashop-ordering`, e por que separamos as portas em dois grupos: `ports/in` e `ports/out`.

---

## 1. O problema que a Arquitetura Hexagonal resolve

Antes de falar de portas, vale lembrar o que a arquitetura hexagonal (também chamada de **Ports & Adapters**, proposta por Alistair Cockburn em 2005) tenta resolver:

> Como manter o **núcleo da aplicação** (regras de negócio, casos de uso) **independente** das tecnologias usadas para entregar e persistir dados (REST, gRPC, JPA, Kafka, S3, etc.)?

A resposta é: o núcleo **não conversa diretamente com nenhuma tecnologia**. Ele expõe e exige **contratos** (interfaces) — as chamadas **portas**. As tecnologias entram no sistema através de **adaptadores** que implementam essas portas.

Visualmente:

```
              ┌──────────── ADAPTADORES PRIMÁRIOS ────────────┐
              │   REST Controllers · Kafka Listener · CLI      │
              └────────────────────┬──────────────────────────┘
                                   │  chamam
                                   ▼
                          ┌────────────────┐
                          │   ports/in     │  ◀── contrato de ENTRADA
                          └────────┬───────┘
                                   │  implementado por
                                   ▼
            ┌─────────────────────────────────────────────────┐
            │             NÚCLEO (application + domain)       │
            │      Casos de uso · Entidades · Regras          │
            └────────────────────┬────────────────────────────┘
                                 │  depende de
                                 ▼
                          ┌────────────────┐
                          │   ports/out    │  ◀── contrato de SAÍDA
                          └────────┬───────┘
                                   │  implementado por
                                   ▼
              ┌──────────── ADAPTADORES SECUNDÁRIOS ──────────┐
              │   JPA Repository · HTTP Client · Mensageria   │
              └───────────────────────────────────────────────┘
```

A regra de ouro: **as setas sempre apontam para dentro do hexágono**. O núcleo nunca depende de tecnologia, é a tecnologia que se pluga no núcleo.

---

## 2. O que é uma Porta (Port)?

Uma **porta** é simplesmente uma **interface Java** que descreve uma capacidade — sem dizer nada sobre como ela é implementada.

Existem dois tipos:

| Tipo | Outros nomes | Quem implementa | Quem consome |
|------|--------------|-----------------|--------------|
| **Port IN** (`ports/in`) | Driving port, Inbound port, API | A camada de aplicação | Adaptadores primários (Controllers, Listeners, CLI) |
| **Port OUT** (`ports/out`) | Driven port, Outbound port, SPI | Adaptadores secundários (JPA, HTTP client) | A camada de aplicação |

> 🔧 **O código só passou a concordar com esta tabela na Fase 16.** Até então os clients HTTP do `product-catalog` e da Rapidex moravam em `infrastructure/adapters/**in**/web/…`, ao lado dos controllers — implementando portas de **saída** a partir da pasta de **entrada**.
>
> Compilava e funcionava; o que se perdia era a única coisa que a estrutura de pastas oferece de graça, que é responder "quem chama quem" sem abrir arquivo. Um leitor novo olhava `adapters/in` e via, juntos, o que o mundo pede ao domínio e o que o domínio pede ao mundo — exatamente a distinção que a arquitetura existe para tornar óbvia.
>
> Hoje estão em `adapters/out/web/{product,shipping}/client/`, ao lado de `adapters/out/persistence/`. A regra é simples: **quem inicia a chamada decide a pasta.** Controller recebe (`in`); client HTTP chama (`out`), mesmo falando o mesmo protocolo.

A diferença está em **quem dirige a conversa**:

- Em uma porta **IN**, o mundo externo **dirige** a aplicação. Um Controller diz: "execute este caso de uso".
- Em uma porta **OUT**, a aplicação **dirige** o mundo externo. O caso de uso diz: "preciso desse dado, alguém me obtenha".

---

## 3. Ports IN — "O que a aplicação OFERECE"

São o **vocabulário do caso de uso**. Cada porta de entrada descreve uma capacidade que a aplicação expõe para o mundo.

Boas práticas de nomeação no projeto: começam com `For...` (em inglês, lê-se como propósito).

```java
// src/main/java/com/gtech/algashop/core/ports/in/shoppingcart/ForQueryShoppingCarts.java
public interface ForQueryShoppingCarts {
    ShoppingCartOutput findById(UUID shoppingCartId);
    ShoppingCartOutput findByCustomerId(UUID customerId);
}
```

Lê-se: **"para consultar carrinhos de compra"**. Esta é a API pública do caso de uso.

### Quem implementa

A camada de **aplicação**, normalmente um `*ApplicationService` ou `*QueryService`:

```java
@Service
@RequiredArgsConstructor
public class ShoppingCartQueryService implements ForQueryShoppingCarts {

    private final ForObtainingShoppingCarts forObtainingShoppingCarts; // porta OUT

    @Override
    public ShoppingCartOutput findById(UUID shoppingCartId) {
        // aqui moram: autorização, cache, orquestração, eventos, regras
        return forObtainingShoppingCarts.findById(shoppingCartId);
    }
}
```

### Quem consome

Os **adaptadores primários** — qualquer coisa que "entra" na aplicação:

```java
@RestController
@RequestMapping("/api/v1/shopping-carts")
@RequiredArgsConstructor
public class ShoppingCartController {

    private final ForQueryShoppingCarts forQueryShoppingCarts; // ← consome a porta IN

    @GetMapping("/{id}")
    public ShoppingCartOutput findById(@PathVariable UUID id) {
        return forQueryShoppingCarts.findById(id);
    }
}
```

O mesmo `ForQueryShoppingCarts` poderia ser consumido por:

- Outro `Controller` REST.
- Um `@KafkaListener` reagindo a um evento.
- Um job agendado (`@Scheduled`).
- Um teste de integração que mocka apenas a infraestrutura.

Nenhum deles precisa saber como o carrinho é buscado.

---

## 4. Ports OUT — "O que a aplicação PRECISA"

São o **vocabulário das dependências externas**. Cada porta de saída descreve algo que o caso de uso exige da infraestrutura para funcionar.

```java
// src/main/java/com/gtech/algashop/core/ports/out/shoppingcart/ForObtainingShoppingCarts.java
public interface ForObtainingShoppingCarts {
    ShoppingCartOutput findById(UUID shoppingCartId);
    ShoppingCartOutput findByCustomerId(UUID customerId);
}
```

Lê-se: **"para obter carrinhos de compra (de algum lugar)"**. O "algum lugar" é responsabilidade do adaptador.

### Quem implementa

Os **adaptadores secundários**, vivendo em `infrastructure/`:

```java
@Component
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class ShoppingCartQueryServiceImpl implements ForObtainingShoppingCarts {

    private final ShoppingCartJpaEntityRepository repository; // detalhe JPA
    private final Mapper mapper;

    @Override
    public ShoppingCartOutput findById(UUID shoppingCartId) {
        ShoppingCartPersistenceEntity entity = repository.findById(shoppingCartId)
                .orElseThrow(ShoppingCartNotFound::new);
        return mapper.convert(entity, ShoppingCartOutput.class);
    }
}
```

O núcleo **não importa** `JpaRepository`, `ShoppingCartPersistenceEntity` ou qualquer coisa de Spring Data. Tudo isso vive atrás da interface.

### Quem consome

A própria **camada de aplicação**, que recebe a porta via injeção e a invoca.

---

## 5. "Mas IN e OUT têm os mesmos métodos. Por que não usar só um?"

Essa é a dúvida natural quando o caso de uso é trivial. A resposta tem três camadas:

### 5.1 Conceitos opostos, ainda que código parecido

| Pergunta | Resposta |
|----------|----------|
| **O que a aplicação OFERECE para o mundo?** | `ports/in` |
| **O que a aplicação PRECISA do mundo?** | `ports/out` |

São duas direções diferentes — apenas coincide hoje que ambas mencionam "buscar carrinho", porque o caso de uso é uma consulta direta. Em um caso de uso de **escrita** (ex.: `ForCheckingOut`), o `in` recebe um `CheckoutInput` e dispara várias portas `out` (cobrança, estoque, persistência). Aí a diferença explode.

### 5.2 Evolução natural do caso de uso

Hoje a implementação é um pass-through. Mas o dia em que aparecer:

```java
@Override
public ShoppingCartOutput findById(UUID cartId) {
    authorizationService.ensureCanRead(cartId);   // autorização

    return cache.get(cartId, () -> {              // cache
        ShoppingCartOutput cart = forObtainingShoppingCarts.findById(cartId);
        cart = priceEnricher.refresh(cart);       // enriquecimento (outra port OUT)
        eventPublisher.publish(new ShoppingCartViewed(cartId)); // evento
        return cart;
    });
}
```

…nada disso pertence ao Controller (que só sabe de HTTP) nem ao adaptador JPA (que só sabe de persistência). Pertence ao caso de uso — e o caso de uso vive atrás da `ports/in`.

Se o Controller chamasse `ports/out` direto, seria preciso refatorar **todos os Controllers** para introduzir essa camada. Mantendo a porta IN desde o início, a evolução é interna ao caso de uso e **transparente** para quem consome.

### 5.3 Inversão de dependência

A regra: **o núcleo não conhece a infraestrutura**.

Se o Controller chamasse `ForObtainingShoppingCarts` diretamente, a borda de entrada (web) estaria acoplada à abstração da borda de saída (persistência). Dois extremos do hexágono se conhecendo, pulando o miolo — exatamente o que a arquitetura tenta evitar.

---

## 6. Estrutura de pacotes no projeto

```
com.gtech.algashop.core
├── application
│   └── shoppingcart
│       └── ShoppingCartQueryService.java       ← implementa ports/in, usa ports/out
├── domain
│   └── model
│       └── shoppingcart                         ← entidades, value objects, regras
└── ports
    ├── in
    │   └── shoppingcart
    │       ├── ForQueryShoppingCarts.java       ← contrato de entrada
    │       └── ShoppingCartOutput.java          ← DTO de resposta
    └── out
        └── shoppingcart
            └── ForObtainingShoppingCarts.java   ← contrato de saída

com.gtech.algashop.infrastructure
└── persistence
    └── shoppingcart
        └── ShoppingCartQueryServiceImpl.java   ← implementa ports/out (JPA)

com.gtech.algashop.presentation
└── shoppingcart
    └── ShoppingCartController.java             ← consome ports/in (REST)
```

Olhando os imports de qualquer classe você consegue dizer em que camada ela vive — e nunca verá `core` importando `infrastructure`.

---

## 7. Fluxo completo de uma requisição

Acompanhe um `GET /api/v1/shopping-carts/{id}`:

```
1. HTTP chega no  ShoppingCartController                    (presentation)
                            │
                            ▼ chama  ForQueryShoppingCarts.findById(id)
2. Spring injeta a impl:  ShoppingCartQueryService           (application)
                            │
                            ▼ (aqui poderia ter: auth, cache, eventos…)
                            │
                            ▼ chama  ForObtainingShoppingCarts.findById(id)
3. Spring injeta a impl:  ShoppingCartQueryServiceImpl       (infrastructure)
                            │
                            ▼ usa  ShoppingCartJpaEntityRepository
4. Spring Data executa SQL no Postgres
                            │
                            ▼ devolve PersistenceEntity
5. Mapper converte para  ShoppingCartOutput (DTO)
                            │
                            ▲ caminho de volta, sem que ninguém saiba a origem real
6. Controller serializa em JSON  →  cliente HTTP
```

O Controller só conhece `ports/in`. O `QueryService` só conhece `ports/out`. Cada um vê o que lhe interessa.

---

## 8. Convenções de nomenclatura adotadas

| Convenção | Exemplo | Propósito |
|-----------|---------|-----------|
| `For...` em `ports/in` | `ForQueryShoppingCarts`, `ForCheckingOut`, `ForManagingCustomers` | Lê-se como "para fazer X" — descreve o caso de uso. |
| `For...` em `ports/out` | `ForObtainingShoppingCarts`, `ForChargingCreditCards`, `ForSendingEmails` | Lê-se como "para obter/cobrar/enviar X" — descreve a dependência. |
| Sufixo `Input` / `Output` | `CheckoutInput`, `ShoppingCartOutput` | DTOs que cruzam as portas — neutros em relação a tecnologia (sem `@Entity`, sem `@JsonProperty`). |
| Implementações na aplicação | `ShoppingCartQueryService`, `CheckoutApplicationService` | Sufixo `Service` para a classe que orquestra o caso de uso. |
| Implementações em infra | `ShoppingCartQueryServiceImpl`, `StripeCreditCardGateway` | Sufixo `Impl` ou nome da tecnologia (Stripe, Postgres) para deixar claro que é adaptador. |

---

## 9. Benefícios concretos no dia a dia

| Benefício | Exemplo prático |
|-----------|-----------------|
| **Testabilidade** | Em testes de `ShoppingCartQueryService`, basta mockar `ForObtainingShoppingCarts` — sem subir banco, Spring, JPA. |
| **Trocar tecnologia** | Migrar de JPA para MongoDB exige apenas uma nova implementação de `ForObtainingShoppingCarts`. Núcleo intacto. |
| **Múltiplos canais de entrada** | O mesmo `ForCheckingOut` pode ser disparado por REST, mensageria e CLI sem duplicar lógica. |
| **Composição de fontes** | Um caso de uso pode declarar várias portas OUT (`ForObtainingShoppingCarts`, `ForFetchingStock`, `ForChargingCreditCards`) e orquestrá-las. |
| **Limite claro para regra de negócio** | Toda lógica vive nos `*Service` que implementam `ports/in`. Controllers ficam burros (só HTTP) e adaptadores também (só tecnologia). |

---

## 10. O que NÃO confundir

| | Port IN | Port OUT |
|---|---------|----------|
| **Direção** | Mundo → Aplicação | Aplicação → Mundo |
| **Quem implementa** | Camada de aplicação | Adaptador secundário (infra) |
| **Quem consome** | Adaptador primário (web, listener, CLI) | Camada de aplicação |
| **Vocabulário** | Caso de uso / regra de negócio | Dependência técnica / capacidade externa |
| **Analogia** | API pública da aplicação | SPI (provider interface) interna |
| **Pode ter regra de negócio?** | Sim, na implementação | Não — adaptadores só traduzem |

- **Porta não é DAO**: um DAO é um detalhe de implementação. A porta descreve **propósito** ("obter carrinhos"), não tecnologia ("buscar via JDBC").
- **Porta não precisa ser 1:1 com tabela**: uma única porta `ForObtainingShoppingCarts` pode juntar dados de várias tabelas, ou até de várias fontes (banco + cache + API).
- **Porta IN e Porta OUT podem ter assinaturas idênticas em casos de uso triviais** — isso é normal e **não é duplicação**, são contratos com propósitos opostos.
- **Não exponha entidades JPA nas portas**: o que cruza a porta são DTOs (`*Input`/`*Output`) que pertencem ao núcleo, não à infraestrutura.

---

## Referências

- Alistair Cockburn — *Hexagonal Architecture* (2005)
- Vaughn Vernon — *Implementing Domain-Driven Design*, capítulo 4 ("Architecture")
- Tom Hombergs — *Get Your Hands Dirty on Clean Architecture* (livro inteiro dedicado ao tema)
