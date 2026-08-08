# Resiliência: os cinco padrões, e as duas respostas diferentes

> Timeout, retry, bulkhead, circuit breaker e fallback — o que cada um resolve, a ordem em que se aninham, e por que o `ordering` e o `billing` chegaram a decisões opostas para o mesmo problema.
> Código real: `infrastructure/config/resilience/SpringCircuitBreakerConfig.java` e `adapters/out/web/**/Resilient*APIClient.java` (ordering); `infrastructure/resilience/SpringCircuitBreakerConfig.java` e `infrastructure/**/Resilient*APIClient.java` (billing).

> Este documento fecha uma pendência registrada desde a Fase 6: *"Resiliência — circuit breaker e retry nas chamadas entre serviços"*.

---

## O problema

Até aqui as chamadas entre serviços eram otimistas. `ordering → product-catalog`, `ordering → Rapidex`, `billing → FastPay`: nenhuma tinha timeout, nenhuma tinha limite de concorrência, e nenhuma parava de tentar quando o outro lado já estava claramente morto.

O detalhe que torna isso pior do que parece:

> **Uma dependência lenta é mais perigosa que uma dependência morta.** A morta recusa a conexão em milissegundos e a thread volta. A lenta segura a thread — e como o pool é finito, um serviço que demora 30 segundos consome, em 30 segundos de tráfego normal, todas as threads que você tem. O seu serviço cai junto, e o log não vai dizer que a culpa foi de outro.

É a falha em cascata: o modo mais comum de um sistema distribuído morrer inteiro por causa de um pedaço.

---

## Os cinco, na ordem em que se aninham

A ordem importa mais que os padrões isolados, porque é ela que explica por que cada um sozinho não basta:

```
   requisição
       │
       ▼
  ┌─ BULKHEAD ────────────────────────────────┐   quantas threads podem entrar
  │  ┌─ CACHE ─────────────────────────────┐  │
  │  │  ┌─ CIRCUIT BREAKER ─────────────┐  │  │   deixo sair para a rede?
  │  │  │  ┌─ RETRY ─────────────────┐  │  │  │   tento de novo?
  │  │  │  │  ┌─ TIMEOUT ─────────┐  │  │  │  │   até quando espero?
  │  │  │  │  │       rede        │  │  │  │  │
  │  │  │  │  └───────────────────┘  │  │  │  │
  │  │  │  └─────────────────────────┘  │  │  │
  │  │  └─────────────────────────────────┘  │  │
  │  └───────────────────────────────────────┘  │
  └─────────────────────────────────────────────┘
                                    │
                                    ▼
                              FALLBACK (se houver)
```

Lida de fora para dentro: o bulkhead decide **quantos** entram, o cache evita a ida, o circuito decide **se** vale sair, o retry decide **quantas vezes**, e o timeout decide **até quando**.

### Timeout — o mais importante dos cinco

E o motivo só fica claro no desenho acima:

> **Sem timeout, o circuito nunca abre.** Um circuit breaker reage a falhas que **terminaram**. Uma chamada pendurada não é falha — é uma chamada em andamento. Ela não conta para nada, não abre nada, e segura a thread para sempre.

```java
// ordering — ProductCatalogApiConfig / RapiDexAPIClientConfig
connectTimeout = 3s     readTimeout = 7s
```

```java
// billing — FastpayPaymentAPIClientConfig
connectTimeout = 3s     readTimeout = 20s
```

Os 20 segundos do pagamento **não** são desleixo — são a decisão mais interessante da configuração inteira:

> Estourar o timeout **não cancela a cobrança do outro lado**. Só desiste de esperar a resposta. Um gateway leva segundos mesmo (antifraude, autorização do emissor), então cortar cedo não protege de nada: só aumenta a frequência do pior estado possível, que é **"não sei se cobrou"**.

O default do JDK, para quem não configura nada, é **infinito**.

### Retry — e o `includes` como decisão de negócio

```java
RetryPolicy.builder()
        .maxRetries(3)          // 3 além da original = 4 idas
        .delay(3s).multiplier(2)  // 3s → 6s → 12s
        .includes(GatewayTimeoutException.class,
                  BadGatewayException.ServerErrorException.class)
        .build();
```

O `includes` é a lista do que vale repetir, e ela é decisão de negócio disfarçada de configuração:

| Falha | Retenta? | Por quê |
|---|---|---|
| Timeout | **sim** | pode ter sido pico momentâneo |
| 5xx | **sim** | problema do outro lado, pode passar |
| 4xx | **não** | repetir um 401 dá 401 quatro vezes |
| 404 | **não** | vira `Optional.empty()` antes de chegar aqui |

Foi para isso que `BadGatewayException` ganhou as subclasses `ServerErrorException` e `ClientErrorException`: o retry casa por **assignability**, então listar só a de servidor deixa os 4xx passando direto.

O **backoff exponencial** é cortesia com quem já está mal. Retentar imediatamente três vezes é somar carga a um serviço em dificuldade — a forma mais rápida de transformar uma degradação em queda.

### Bulkhead — o compartimento estanque

O nome vem dos navios: o casco é dividido em compartimentos para que um furo não afunde o barco inteiro.

```java
@ConcurrencyLimit(10)   // catálogo e Fastpay
@ConcurrencyLimit(15)   // Rapidex
```

Limita quantas threads podem estar dentro daquele método ao mesmo tempo. Se o catálogo travar, no máximo 10 threads ficam presas — as outras continuam servindo carrinho, pedido e cliente.

> ⚠️ **O `@ConcurrencyLimit` do Spring é bloqueante, não fail-fast.** A décima primeira thread **espera** na fila; ela não recebe erro imediato. Isso protege a dependência, e protege o seu serviço só até a fila crescer. Um `Semaphore` com `tryAcquire` e timeout falharia rápido — é outra escolha, com outro custo.

E vale saber a conta do pior caso: 4 tentativas × 7s de read timeout + 21s de backoff ≈ **49 segundos** com uma vaga ocupada.

### Circuit breaker — o único que age antes de a chamada sair

Os outros quatro reagem depois de tentar. O circuito é o único que decide **não tentar**.

```
CLOSED ──── falha ────► OPEN ──── openTimeout ────► HALF_OPEN
   ▲                                                    │
   └──────────────── sucesso ───────────────────────────┘
                                                        │
                              falha ────────────────────┘  (volta a OPEN)
```

`HALF_OPEN` é o que permite voltar sozinho: passado o tempo de espera, **uma** chamada é deixada passar como teste. Deu certo, fecha; falhou, reabre. Sem esse estado, alguém teria que reiniciar o serviço para o circuito voltar.

Aqui o circuito é usado **programaticamente**, não por anotação:

```java
this.circuitBreaker = (FrameworkRetryCircuitBreaker)
        circuitBreakerFactory.create("productCatalogCB");
// ...
return circuitBreaker.run(() -> loadProduct(productId));
```

> ⚠️ **Não há limiar de falhas.** **Uma única** execução que termine em exceção leva CLOSED → OPEN. É uma diferença grande em relação ao Resilience4j, cujo padrão é abrir por **percentual de falha numa janela deslizante**. Aqui um pico isolado abre o circuito.

### Fallback — a pergunta que ele obriga a responder

*Existe resposta aproximada aceitável?* E a resposta honesta quase sempre é "depende do que a resposta significa".

---

## A assimetria: dois clients, duas respostas

É o coração desta fase. Os dois clients do `ordering` foram escritos na mesma leva, pela mesma pessoa, com a mesma biblioteca — e decidiram o oposto.

```java
// frete — TEM fallback
DeliveryCostResponse response = circuitBreaker.run(
        () -> doCalculate(request),
        ex -> doInternalFallback(request, ex)   // R$ 20,00 em 10 dias
);
```

```java
// catálogo — NÃO tem
return circuitBreaker.run(() -> loadProduct(productId));
// falha vira 502 ou 504
```

| | frete | produto |
|---|---|---|
| Resposta aproximada existe? | **sim** — uma estimativa é plausível | **não** — não dá para inventar preço |
| Custo de errar | prejuízo controlado na margem | pedido inválido, cobrança errada |
| Falha vira | valor estimado, silenciosamente | 502 / 504 |

A regra que se extrai: **fallback só é honesto quando a resposta aproximada é aceitável para o negócio, não quando ela é conveniente para o código.** Frete estimado por baixo custa margem; preço de produto inventado custa a venda inteira e a confiança.

> ⚠️ **O que esse fallback custa, dito por inteiro.** Como o `circuitBreaker.run` recebe a função de fallback, a exceção **nunca escapa** — o `unwrapException` daquele arquivo é código inalcançável, e uma queda da transportadora **nunca vira 502 ou 504**. O cliente recebe um preço que não veio de lugar nenhum, sem saber disso. Há um `log.warn`, e é só ele que separa "estimativa" de "mentira": se ninguém observar esse log, o sistema mente em silêncio e ninguém descobre.
>
> Foi mantido como está, e a decisão é defensável — mas ela é de **produto**, não de engenharia, e merecia estar escrita em algum lugar que a área de negócio leia.

---

## A decisão que atravessa tudo: idempotência

O `billing` levou a mesma pergunta a outro nível. Ele tem **dois circuitos para o mesmo host**, separados não por dependência, mas por **o que a operação faz**:

```java
fastpayCB       // findByCode, findById de cartão      COM retry
fastpayWriteCB  // capture, create/delete de cartão    maxRetries(0)
```

O motivo cabe em uma frase:

> **Timeout num POST de cobrança não cancela a autorização do outro lado.** Significa *"não sei se cobrou"* — e repetir é o caminho clássico da cobrança dupla.

Quem resolve a incerteza não é outra tentativa; é a **conciliação**: o webhook do gateway, ou uma consulta por `findByCode`. Retentar transforma uma dúvida em um problema.

Por que não um circuito só, traduzindo as exceções do `capture` para tipos fora do `includes`? Funcionaria — e o "não retenta" passaria a depender de um detalhe silencioso da tradução. Qualquer mudança futura ali reativaria o retry na cobrança sem ninguém perceber. **Em código que cobra dinheiro, explícito ganha de esperto.**

O custo está registrado: dois estados independentes para a mesma dependência. Se o FastPay cair, os dois abrem separadamente, cada um na sua primeira falha.

A mesma régua vale para o cartão: `create` não é idempotente (retentar cria tokens duplicados), `delete` é — mas ficou no circuito de escrita mesmo assim, porque o registro local já foi apagado antes da chamada e repetir não tem ganho.

---

## O que resiliência não resolve

A mudança mais importante do `billing` nesta leva não é nenhum dos cinco padrões:

```java
// application/invoice/management/InvoicePaymentTransactions.java
loadPaymentRequest (tx)  →  capture (SEM tx)  →  assignPayment (tx)
```

Antes, `processPayment` era `@Transactional` e chamava o gateway lá dentro. A conexão do pool ficava presa durante toda a chamada HTTP. Com o gateway lento, dez faturas simultâneas esgotavam o pool e derrubavam o billing **inteiro** — inclusive o `GET` de fatura, que nem toca no gateway.

> **Nenhum circuit breaker corrige isso.** Quando a chamada chega ao breaker, a conexão do banco **já foi adquirida** pelo `@Transactional`, que está acima dele na pilha. O breaker protege o que está abaixo dele; o recurso foi tomado acima.
>
> A lição é maior que o caso: **resiliência não é uma camada que se adiciona por cima.** É uma propriedade de como os recursos escassos — threads, conexões, memória — são adquiridos e liberados. Padrão nenhum salva um desenho que segura um recurso caro enquanto espera a rede.

A classe é separada, e não métodos privados do application service, porque autoinvocação não passa pelo proxy do Spring — `@Transactional` seria ignorado.

Pelo mesmo motivo, o `CreditCardManagementService` perdeu o `@Transactional` de classe, e o `ShippingApplicationService` nasceu sem nenhum.

---

## Armadilhas

1. **Sem timeout, o circuito nunca abre.** É o padrão que habilita os outros.
2. **O default do JDK é esperar para sempre.** Não configurar é escolher infinito.
3. **`@ConcurrencyLimit` é bloqueante.** Ele enfileira, não recusa.
4. **Retentar 4xx é desperdício garantido.** O `includes` é o que separa.
5. **Retentar POST que cobra dinheiro é cobrança dupla.** Idempotência decide, não o código de erro.
6. **Fallback silencioso mente sem avisar.** Só o log separa estimativa de invenção.
7. **Chamada HTTP dentro de transação esgota o pool**, e nenhum breaker corrige.
8. **Estado de circuito vaza entre testes** — é singleton do contexto, e o contexto é compartilhado.

---

## Pendências registradas

- [ ] **`openTimeout` e o backoff não conversam.** O circuito fica aberto 5s, mas um ciclo de retry esgotado leva 21s só de backoff, mais o read timeout de cada tentativa. Quando o circuito abre, o tempo dele já expirou — então a chamada seguinte tende a passar como HALF_OPEN em vez de falhar rápido. Na prática o circuito protege menos do que a configuração sugere.
- [ ] **Uma falha abre o circuito.** Sem limiar nem janela, um pico isolado corta a dependência inteira por 5 segundos.
- [ ] **O fallback do frete nunca deixa a falha virar erro.** Nenhuma métrica conta quantas respostas foram estimadas.
- [ ] **`@ConcurrencyLimit` é por método, não por dependência.** No `billing` são 7 limitadores independentes de 10 permits cada — não um bulkhead de 10 para o FastPay. Duas operações diferentes podem, somadas, ocupar 20 threads.
- [ ] **Dois circuitos abrem separados para o mesmo host.** Uma queda total do FastPay custa no mínimo duas falhas reais antes de qualquer proteção.
- [ ] **O teste de bulkhead não satura o bulkhead.** O `ProductCatalogServiceIT` faz 6 chamadas contra um limite de 10 — nenhuma thread chega a esperar.
- [x] ~~**Nenhuma métrica observa os circuitos.**~~ Resolvido na Fase 17: o `/actuator/health` passou a expor o estado de cada circuito, verificado abrindo um de verdade. Continua sem **contador** de aberturas — o endpoint mostra o estado agora, não o histórico —, e o status `DEGRADED` que ele produz devolve **HTTP 200**, então um probe automático não distingue circuito aberto de serviço saudável. Ver [`health-checks.md`](../04-infraestrutura/health-checks.md).
- [ ] **Sem outbox nem reconciliação automática.** A fatura pendente depende do webhook do gateway ou de alguém consultar.

---

## Checklist de revisão

- [ ] Sei explicar por que uma dependência lenta é pior que uma morta
- [ ] Sei desenhar a ordem em que os cinco padrões se aninham
- [ ] Entendo por que sem timeout o circuit breaker nunca abre
- [ ] Sei por que 4xx não deve ser retentado e 5xx deve
- [ ] Entendo a diferença entre bulkhead bloqueante e fail-fast
- [ ] Sei o que HALF_OPEN resolve
- [ ] Sei dizer quando um fallback é honesto e quando ele mente
- [ ] Entendo por que `capture` não pode ser retentado, e o que resolve a incerteza no lugar
- [ ] Sei por que dois circuitos para o mesmo host podem ser a escolha certa
- [ ] Entendo por que chamada HTTP dentro de transação não é problema que breaker resolva

---

## Referências

- [Release It! — Michael Nygard](https://pragprog.com/titles/mnee2/release-it-second-edition/) — a origem de circuit breaker e bulkhead como padrões nomeados
- [Spring Cloud Circuit Breaker](https://docs.spring.io/spring-cloud-circuitbreaker/reference/)
- [Spring Framework — Resilience features](https://docs.spring.io/spring-framework/reference/core/resilience.html)
- [`resiliencia-config.md`](../04-infraestrutura/resiliencia-config.md) — os parâmetros, a biblioteca e como testar
- [`cache.md`](./cache.md) — a camada que fica entre o bulkhead e o circuito
- [`tratamento-erros-api.md`](../03-testes-integracao/tratamento-erros-api.md) — 502 e 504, e por que ganharam subclasses
- [`ports-hexagonal.md`](./ports-hexagonal.md) — por que os clients HTTP são adaptadores de saída
