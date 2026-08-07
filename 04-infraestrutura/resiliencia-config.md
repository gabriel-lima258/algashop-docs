# Resiliência na prática: configuração e teste

> Onde cada parâmetro mora, quais são os valores reais por cliente, a biblioteca que quase ninguém documenta ainda — e o padrão de teste que distingue "retentou" de "não retentou".
> Código real: `SpringCircuitBreakerConfig.java` e `SpringResilienceConfig.java` (nos dois serviços), `*APIClientConfig.java`, `ResilientProductCatalogAPIClientIT.java`, `FastpayResilienceIT.java`.

> Os padrões e as decisões de desenho estão em [`resiliencia.md`](../01-arquitetura-design/resiliencia.md). Aqui é a máquina.

---

## A biblioteca: não é Resilience4j

Vale abrir por aqui, porque é a primeira coisa que confunde quem procura material.

| O que se procura | O que está no código |
|---|---|
| `io.github.resilience4j` | `org.springframework.cloud:spring-cloud-starter-circuitbreaker-framework-retry` |
| `@CircuitBreaker`, `@Retry`, `@Bulkhead` | `circuitBreaker.run(...)` programático + `@ConcurrencyLimit` |
| `resilience4j.circuitbreaker.instances.*` no YAML | `Customizer<FrameworkRetryCircuitBreakerFactory>` em Java |
| `slidingWindowSize`, `failureRateThreshold` | `openTimeout`, `resetTimeout` — **não existe janela nem limiar** |

São duas peças distintas:

**1. Spring Cloud CircuitBreaker, implementação `framework-retry` (5.0.0)** — circuito e retry **acoplados**: a `RetryPolicy` faz parte da configuração do breaker. Construída sobre `org.springframework.core.retry`, que é novo do **Spring Framework 7**.

**2. Resiliência nativa do Spring Framework 7** — `@EnableResilientMethods` liga o pós-processamento, e `@ConcurrencyLimit` é o bulkhead. Não precisa de dependência nenhuma: vem no `spring-context`.

```java
@Configuration
@EnableResilientMethods
public class SpringResilienceConfig { }
```

Onze linhas, e sem elas o `@ConcurrencyLimit` é ignorado — silenciosamente, como toda anotação sem o `@Enable` correspondente.

> Praticamente todo tutorial, curso e resposta de fórum sobre resiliência em Spring ensina Resilience4j. Copiar de lá aqui não dá erro de compilação óbvio: dá dependência que não resolve, ou anotação que não faz nada.

---

## Onde cada parâmetro mora

### Timeout — no cliente HTTP, não na política

É o único dos cinco que não passa pelo circuit breaker. Fica na fábrica de requisições do `RestClient`:

```java
// billing — FastpayPaymentAPIClientConfig
HttpClient httpClient = HttpClient.newBuilder()
        .connectTimeout(Duration.ofSeconds(3))
        .build();

JdkClientHttpRequestFactory factory = new JdkClientHttpRequestFactory(httpClient);
factory.setReadTimeout(Duration.ofSeconds(20));
```

| Cliente | Connect | Read | Por quê |
|---|---|---|---|
| `product-catalog` (ordering) | 3s | 7s | consulta de leitura, pode falhar rápido |
| Rapidex (ordering) | 3s | 7s | idem |
| FastPay cartão (billing) | 3s | 10s | validação com a bandeira leva mais |
| FastPay pagamento (billing) | 3s | **20s** | cortar cedo só aumenta o "não sei se cobrou" |

> **`JdkClientHttpRequestFactory` no lugar do `SimpleClientHttpRequestFactory`**, e não é preciosismo. O `Simple` usa `HttpURLConnection`, que **não tem pool de conexões** — handshake TLS inteiro a cada chamada contra host externo — e, pior para este assunto, **reenvia GETs sozinho** em falha de I/O. Um retry invisível debaixo do retry configurado, que ninguém conta e nenhum teste vê.

### Circuit breaker e retry — em Java, com defaults por property

```java
@Bean
public Customizer<FrameworkRetryCircuitBreakerFactory> defaultCustomizer(
        @Value("${algashop.resilience.circuit-breaker.max-retries:3}") long maxRetries,
        @Value("${algashop.resilience.circuit-breaker.delay:3s}") Duration delay,
        @Value("${algashop.resilience.circuit-breaker.multiplier:2}") double multiplier,
        @Value("${algashop.resilience.circuit-breaker.open-timeout:5s}") Duration openTimeout,
        @Value("${algashop.resilience.circuit-breaker.reset-timeout:30s}") Duration resetTimeout) {
```

| Parâmetro | Valor | O que faz |
|---|---|---|
| `max-retries` | 3 | tentativas **além** da original — 4 idas no total |
| `delay` | 3s | espera antes da primeira retentativa |
| `multiplier` | 2 | dobra a cada rodada: 3s → 6s → 12s |
| `open-timeout` | 5s | quanto tempo o circuito fica aberto antes de testar HALF_OPEN |
| `reset-timeout` | 30s | tempo **sem falha** que zera o estado |

Os cinco passaram a estar declarados em `application-base` nesta leva. Antes existiam apenas como default dentro do `@Value` — a política que governa produção não aparecia em nenhum arquivo que um operador fosse ler.

### As instâncias

| Serviço | Instância | Política |
|---|---|---|
| ordering | `productCatalogCB` | retry completo |
| ordering | `rapidexAPICB` | retry completo |
| billing | `fastpayCB` | retry completo — **leituras** |
| billing | `fastpayWriteCB` | `maxRetries(0)` — **escritas** |

O `fastpayWriteCB` é construído com `maxRetries(0)` **fixo no código**, ignorando a property. É deliberado: a decisão de não repetir uma cobrança não pertence a um arquivo de configuração, onde alguém pode mudá-la sem entender o que está mudando.

### Bulkhead — na anotação

```java
@ConcurrencyLimit(10)   // catálogo, FastPay
@ConcurrencyLimit(15)   // Rapidex
```

> ⚠️ **É por método, não por dependência.** Cada método anotado tem seu próprio semáforo. No `billing` são sete limitadores independentes de 10 permits — não um bulkhead de 10 para o FastPay. `capture` e `findByCode` podem, somados, ocupar 20 threads.

---

## Como testar resiliência

O `ResilientProductCatalogAPIClientIT` estabelece o padrão, e ele tem quatro peças que valem copiar.

### 1. Asserção por contagem de requests

É o que separa um teste de resiliência de um teste que só verifica a exceção final:

```java
assertThatThrownBy(() -> client.getById(SERVER_ERROR_PRODUCT))
        .isInstanceOf(BadGatewayException.ServerErrorException.class);
verifyCallCount(SERVER_ERROR_PRODUCT, 4);   // ← a asserção que importa
```

Sem a contagem, esse teste passaria igual com retry desligado. O número **4** é o que prova as 3 retentativas.

E o mesmo mecanismo prova o circuito aberto — pela **ausência** de chamadas:

```java
assertThatThrownBy(() -> client.getById(SERVER_ERROR_PRODUCT))...;
verifyCallCount(SERVER_ERROR_PRODUCT, 4);
// segunda chamada: falha com a MESMA exceção, mas instantânea
assertThatThrownBy(() -> client.getById(SERVER_ERROR_PRODUCT))...;
verifyCallCount(SERVER_ERROR_PRODUCT, 4);   // não subiu: o circuito cortou
```

No `billing` a mesma técnica sustenta a garantia mais importante da suíte:

```java
// uma única tentativa de cobrança
wireMockServer.verify(1, postRequestedFor(urlEqualTo(PAYMENTS_URL)));
```

### 2. Tempos encurtados por property

```java
@TestPropertySource(properties = {
    "algashop.resilience.circuit-breaker.max-retries=3",
    "algashop.resilience.circuit-breaker.delay=20ms",
    "algashop.resilience.circuit-breaker.multiplier=1",
    "algashop.resilience.circuit-breaker.open-timeout=200ms",
    "algashop.resilience.circuit-breaker.reset-timeout=5s"
})
```

Com os valores de produção, **um** teste de retry ficaria 21 segundos parado no backoff. É a razão principal de os cinco parâmetros virem de property em vez de constante — testabilidade primeiro, ajuste por ambiente de bônus.

### 3. Reset do circuito entre classes

```java
private void resetProductCatalogCircuitBreaker() {
    FrameworkRetryCircuitBreaker circuitBreaker =
            (FrameworkRetryCircuitBreaker) circuitBreakerFactory.create("productCatalogCB");
    circuitBreaker.getCircuitBreakerPolicy().reset();
}
```

> ⚠️ **O estado do circuito é um singleton do contexto, e o contexto é compartilhado entre classes de teste.** Uma classe que derruba o WireMock de propósito para testar o 504 deixa o circuito **aberto** — e a próxima classe recebe 504 sem nem chamar o serviço. O sintoma é o clássico "passa sozinho, falha na suíte".

### 4. Injetar caos no stub, não no código

Os cenários vivem em mappings do WireMock:

| Mapping | Simula |
|---|---|
| `fault: EMPTY_RESPONSE` | conexão derrubada no meio |
| `fault: CONNECTION_RESET_BY_PEER` | reset de conexão |
| `fixedDelayMilliseconds: 15000` | dependência lenta — dispara timeout e circuito |
| `status: 500` / `401` / `204` | erro de servidor, não autorizado, corpo vazio |

Durante o desenvolvimento desta leva, o caos chegou a ser injetado **direto no `ProductController` do product-catalog** — um 400 determinístico por UUID e 90% de chance de dormir 20 segundos. Funcionou para exercitar o circuito, e foi removido: código de caos num controller de produção é uma bomba-relógio, e não roda em CI. O stub roda.

> ⚠️ **Porta dinâmica, não fixa.** O IT do ordering sobe o WireMock em porta sorteada, num bloco `static` (porque `@DynamicPropertySource` é avaliado durante o refresh do contexto). O `FastpayResilienceIT` do billing ainda usa a porta fixa **8788**, herdada do `AbstractFastpayImplIT` — e disputa essa porta com os outros ITs de FastPay na mesma JVM.

---

## Onde os testes vivem

```bash
./gradlew test              # NÃO roda nenhum teste desta fase — a task exclui *IT
./gradlew integrationTest   # é aqui
./gradlew check             # test + contractTest + integrationTest
```

Todos os testes de resiliência são `*IT`, e a task `test` filtra `excludeTestsMatching("*IT")`. É fácil rodar `test`, ver verde, e concluir que a resiliência está coberta.

---

## Armadilhas

1. **Procurar Resilience4j e não achar.** A stack é outra, e a documentação disponível é quase toda da outra.
2. **`@EnableResilientMethods` ausente** faz `@ConcurrencyLimit` virar enfeite, sem erro.
3. **`SimpleClientHttpRequestFactory` reenvia GET sozinho** — um retry a mais que ninguém contou.
4. **Não configurar timeout é escolher infinito.**
5. **Estado de circuito vaza entre classes de teste.**
6. **Teste de retry sem contagem de requests não testa retry.**
7. **Porta fixa de WireMock disputa com outros ITs.**
8. **`./gradlew test` não roda nada disto.**

---

## Pendências registradas

- [ ] **Nenhuma observabilidade dos circuitos.** Sem Actuator, sem métrica de abertura, sem contador de retentativas. O estado só aparece em `log.info` antes de cada chamada — o suficiente para depurar um caso, insuficiente para saber se o circuito abre em produção.
- [ ] **`FastpayResilienceIT` em porta fixa 8788**, compartilhando JVM com os outros ITs de FastPay.
- [ ] **`openTimeout` de 5s contra 21s de backoff** — ver [`resiliencia.md`](../01-arquitetura-design/resiliencia.md#pendências-registradas).
- [ ] **Timeouts HTTP hardcoded.** Diferente dos parâmetros do circuito, os `Duration.ofSeconds(...)` dos clientes não vêm de property — mudá-los exige recompilar.
- [ ] **Nada configurado em `docker` e `production`.** Os perfis herdam o `base`, o que hoje funciona; mas um ambiente com latência diferente precisaria de valores próprios, e não há lugar preparado para isso.
- [ ] **`billing-scheduler` chama o FastPay sem nenhuma proteção.** O job de cancelamento usa o client cru — sem timeout configurado, sem circuito, sem bulkhead.

---

## Checklist de revisão

- [ ] Sei dizer qual biblioteca está em uso e por que a documentação de fora não serve
- [ ] Sei onde mora cada um dos cinco parâmetros
- [ ] Entendo por que o read timeout do pagamento é maior que o das leituras
- [ ] Sei por que o `fastpayWriteCB` ignora a property de retries
- [ ] Entendo por que `@ConcurrencyLimit` por método não é um bulkhead por dependência
- [ ] Sei escrever um teste que distingue "retentou" de "não retentou"
- [ ] Sei por que o circuito precisa ser resetado entre classes de teste
- [ ] Sei que `./gradlew test` não exercita nada disto

---

## Referências

- [Spring Cloud Circuit Breaker](https://docs.spring.io/spring-cloud-circuitbreaker/reference/)
- [Spring Framework — Resilience](https://docs.spring.io/spring-framework/reference/core/resilience.html)
- [WireMock — Simulating Faults](https://wiremock.org/docs/simulating-faults/)
- [`resiliencia.md`](../01-arquitetura-design/resiliencia.md) — os padrões e as decisões
- [`stubs-contract-tests.md`](../03-testes-integracao/stubs-contract-tests.md) — WireMock, stubs e contract tests
- [`ambiente-local.md`](./ambiente-local.md) — portas e serviços de apoio
