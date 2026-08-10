# Threads, threads virtuais e onde o sistema realmente satura

> O que aconteceu quando o `ordering` foi medido sob carga: o limite que apertou primeiro não era o esperado, ligar threads virtuais **piorou o sistema em 9×**, e o serviço travou de vez sem nunca voltar.
> Configuração real: `docker-compose.services.yml` (serviço `algashop-ordering`), `SpringCircuitBreakerConfig` e `ResilientProductCatalogAPIClient` do ordering.

> Instrumento e metodologia em [Testes de carga com k6](../03-testes-integracao/testes-de-carga-k6.md).

---

## O modelo thread-per-request

No Tomcat clássico, **uma requisição ocupa uma thread do início ao fim**. Enquanto o `BuyNowApplicationService` espera a resposta HTTP do catálogo, a thread está parada, sem gastar CPU, mas indisponível para qualquer outra requisição.

A capacidade sai da Lei de Little:

```
vazão máxima = threads ÷ tempo de serviço
```

Com 10 threads e 8,6ms por requisição: 10 ÷ 0,0086 ≈ **1160 req/s**. Foi exatamente o que a medição deu.

O desperdício é a parte que dói: dessas 8,6ms, a maior parte é **espera de rede**, não trabalho. Threads de plataforma custam ~1MB de stack cada e são escassas; deixar mil delas paradas esperando I/O é caro. É esse desperdício que as threads virtuais atacam.

---

## O ambiente da medição

```yaml
algashop-ordering:
  deploy:
    resources:
      limits:
        memory: 512M
  environment:
    SERVER_TOMCAT_THREADS_MAX: ${TOMCAT_THREADS_MAX:-10}
    SPRING_THREADS_VIRTUAL_ENABLED: ${VIRTUAL_THREADS:-false}
```

10 threads (o padrão do Boot é **200**) é deliberadamente pouco. Sem apertar isso, uma máquina de desenvolvimento não chega perto de um gargalo e o teste de carga não mede nada.

Alvo: `POST /api/v1/orders`, que abre transação no Postgres, busca o cliente, chama o **product-catalog** por HTTP, chama a **Rapidex** por HTTP e grava o pedido — as duas chamadas de rede **dentro** da transação.

---

## Resultado 1 — o limite que apertou primeiro foi a memória

Antes de qualquer coisa relacionada a thread, o container **morreu**:

```
ExitCode=137  OOMKilled=true
...
algashop-ordering-1  | Killed
```

Com `memory: 256M`, o `ordering` foi morto pelo kernel por volta de **1600 req/s**. Não houve erro de aplicação, não houve stack trace: o processo simplesmente sumiu, e o k6 passou a acusar 76% de falha porque não havia mais ninguém escutando na porta.

> A lição não é "256M é pouco". É que **o limite que você configurou para observar não é necessariamente o que vai apertar primeiro** — e um limite que aperta antes esconde todos os outros. O catálogo já tinha sido subido para 512M numa fase anterior sem que ninguém registrasse o porquê; era o mesmo fenômeno.

Depois de subir o `ordering` para 512M, o experimento pôde continuar.

---

## Resultado 2 — o teto de 10 threads, medido

Rampa de taxa (`ramping-arrival-rate`) de 1600 até 4000 req/s, 512M, threads de plataforma:

| | |
|---|---|
| Vazão sustentada | **1156 req/s** |
| `p(95)` | 2,44s |
| `avg` | 819ms |
| `max` | 5,16s |
| `http_req_failed` | **0%** |
| `dropped_iterations` | 66.637 |

O log confirma o teto de forma direta — só existem dez nomes de thread:

```
[nio-8081-exec-1] ... [nio-8081-exec-10]
```

E a conta fecha: 10 threads ÷ 8,6ms ≈ 1156 req/s. **A vazão foi exatamente o que a Lei de Little previa.**

Repare no formato da degradação: **zero erros**. Nada quebrou. As requisições excedentes ficaram na fila de accept do TCP e foram atendidas mais tarde — daí `p(95)` de 2,44s. O sistema ficou lento, não errado. Isso é saturação *bem-comportada*, e o pool de threads é o que a produz: ele funciona como **controle de admissão**, limitando quantas requisições entram de fato no sistema.

Guarde essa frase. É o que o próximo resultado destrói.

---

## Resultado 3 — threads virtuais pioraram tudo em 9×

Mesma máquina, mesma carga, mesmos 512M. Única mudança:

```bash
VIRTUAL_THREADS=true docker compose up -d --force-recreate algashop-ordering
```

| | Plataforma (10) | Virtuais |
|---|---|---|
| Vazão sustentada | **1156 req/s** | **127 req/s** |
| `p(95)` da iteração | 2,44s | **60s** (timeout) |
| `http_req_failed` | 0% | **21%** |
| Estado final do serviço | saudável | **travado, sem retorno** |

Primeiro, a prova de que a chave virou. Os nomes das threads mudam de dez threads numeradas para milhares:

```
[at-handler-7884]   [at-handler-7850]   [at-handler-7822]  ...
```

> Isso também demonstra a pegadinha de configuração: com `spring.threads.virtual.enabled=true`, o Tomcat troca o pool por um executor de threads virtuais e **`server.tomcat.threads.max` passa a ser ignorado**. A linha continua no compose, sem efeito, sem aviso.

### Onde as requisições ficaram presas

Um thread dump (`kill -3`) durante a carga mostrou o ponto exato:

```
at org.springframework.aop.interceptor.ConcurrencyThrottleInterceptor.invoke(...)
at org.springframework.resilience.annotation.ConcurrencyLimitBeanPostProcessor
        $ConcurrencyLimitInterceptor.invoke(...)
at org.springframework.cache.interceptor.CacheInterceptor.invoke(...)
```

É o **bulkhead** do cliente do catálogo, configurado na Fase 16:

```java
@ConcurrencyLimit(10) // no maximo 10 threads aqui dentro; as demais BLOQUEIAM
```

Com 10 threads no Tomcat, esse limite **nunca podia disparar** — não existiam 11 threads para limitar. Ele era código morto. Tirar o teto do Tomcat foi o que o acordou. E o comentário dele já dizia o problema: as demais **bloqueiam**. O `ConcurrencyThrottleInterceptor` não rejeita e não tem fila limitada — ele faz `wait()` indefinidamente.

### A sequência completa

1. Threads virtuais removem o teto de entrada. Milhares de requisições entram no sistema.
2. Cada uma abre transação e **segura uma conexão do Hikari** (pool nunca declarado — default **10**).
3. Cada uma bate no `@ConcurrencyLimit(10)` do catálogo e bloqueia.
4. Ninguém é rejeitado. A fila de espera é ilimitada e cresce.
5. As requisições passam a estourar 60s no cliente.
6. O serviço **não volta**. Dois minutos com carga zero e continuava sem responder.

```
  t+20s: HTTP 000    t+80s:  HTTP 000
  t+40s: HTTP 000    t+100s: HTTP 000
  t+60s: HTTP 000    t+120s: HTTP 000
```

Nem `/actuator/health` respondeu. O container seguia `Up`, o processo vivo, servindo absolutamente nada.

---

## A conclusão que a medição impõe

Não é "threads virtuais são ruins". É mais desconfortável que isso:

> **O pool de threads era o único controle de admissão do sistema.** Enquanto ele existia, o excesso de carga esperava *fora* da aplicação, na fila do TCP, sem consumir conexão de banco nem semáforo de bulkhead. Removê-lo não aumentou a capacidade — apenas moveu a fila para **dentro**, para um lugar onde cada requisição enfileirada segura recursos escassos.

Threads virtuais **não deixam nada mais rápido**. Elas deixam mais coisas esperarem ao mesmo tempo. Se o que está adiante é limitado a 10 — pool de conexão, bulkhead, o próprio banco —, permitir que 5.000 requisições cheguem lá é trocar uma fila barata por uma fila cara.

### O mapa dos limitadores

| Limitador | Valor | Ativo com threads de plataforma (10) | Ativo com threads virtuais |
|---|---|---|---|
| Threads do Tomcat | 10 | **sim — é o gargalo** | ignorado |
| Pool do Hikari | 10 (default, nunca declarado) | não (10 threads ≤ 10 conexões) | **sim** |
| `@ConcurrencyLimit` do catálogo | 10 | não — inalcançável | **sim — trava aqui** |
| `@ConcurrencyLimit` da Rapidex | 15 | não — inalcançável | possivelmente |
| Memória do container | 512M | apertou primeiro a 256M | — |

Quatro limitadores de valor 10, configurados em três fases diferentes, por três razões diferentes, sem que ninguém tivesse olhado os quatro juntos. É essa a raiz do problema — não a tecnologia de thread.

---

## O `@Transactional` que segura conexão durante I/O

```java
@Transactional
public String buyNow(BuyNowInput input) {
    Customer customer = customers.ofId(customerId)...;
    Product product = findProduct(...);                 // HTTP -> product-catalog
    var result = calculateShippingCost(input.getShipping());  // HTTP -> Rapidex
    orders.add(order);
}
```

A transação abre antes das duas chamadas de rede e só fecha depois. A conexão do Hikari fica presa durante toda a latência de rede — e o pool tem 10.

Em regime normal isso passa despercebido: as chamadas levam poucos milissegundos. Sob threads virtuais, é o que transforma cada requisição enfileirada num consumidor de recurso escasso.

E há um caso muito pior, já configurado: o retry da Fase 16 é de 3 tentativas com backoff 3s → 6s → 12s. **Uma falha do catálogo segura uma das 10 conexões por 21 segundos.** Metade do pool em duas falhas simultâneas.

O `billing` já corrigiu exatamente isso na Fase 16, tirando a chamada HTTP de dentro da transação em `InvoicePaymentTransactions`. O `ordering` não. Fica registrado como pendência, com o número medido — a correção é uma mudança no caminho crítico de compra e merece commit próprio.

---

## Pinning: por que não é o problema aqui

O grande porém histórico das threads virtuais era o *pinning*: dentro de um bloco `synchronized`, a thread virtual ficava presa à thread portadora e não podia ser desmontada. Poucas portadoras + muitas threads presas = deadlock de fato.

Em **Java 25** isso deixou de valer — o [JEP 491](https://openjdk.org/jeps/491) fez `synchronized` deixar de fixar a portadora. Este projeto está em Java 25, e o travamento observado **não** foi pinning: foi fila ilimitada num semáforo de aplicação, que aconteceria igual com threads de plataforma se houvesse 5.000 delas.

Vale registrar porque é a primeira hipótese que todo mundo levanta, e neste caso ela é a errada.

---

## O ponto cego que a Fase 17 já tinha avisado

O serviço travado continuava, para o Docker, perfeitamente saudável: `Up`, sem `HEALTHCHECK` no Dockerfile e sem `healthcheck:` no compose. E o `/actuator/health`, que saberia responder, **também não respondia** — ele depende das mesmas threads.

> Um health check que compartilha o pool de threads da aplicação não consegue reportar exaustão desse pool. Quando a resposta mais importa, ela não chega.

Isso reforça a pendência aberta em [Health check e degradação](./health-checks.md): o endpoint existe e ninguém o consome. Aqui teria dado o alarme — se alguém estivesse ouvindo, e se ele tivesse como responder.

---

## Como reproduzir

```bash
docker compose up -d
k6 run etc/k6/buy-now.js                                   # dentro da capacidade
k6 run -e PROFILE=volume etc/k6/buy-now.js                 # rampa de concorrência
```

Virando a chave:

```bash
VIRTUAL_THREADS=true docker compose up -d --force-recreate algashop-ordering
```

E de volta:

```bash
docker compose up -d --force-recreate algashop-ordering
```

Para confirmar qual executor está servindo, olhe o nome das threads no log: `nio-8081-exec-N` (plataforma, N ≤ `threads.max`) ou `tomcat-handler-N` (virtuais, N sem teto).

---

## Armadilhas

- **`server.tomcat.threads.max` vira letra morta** quando `spring.threads.virtual.enabled=true`. Sem aviso, sem log.
- **`@ConcurrencyLimit` bloqueia, não rejeita.** Sob concorrência alta ele vira uma fila ilimitada. Bulkhead que não rejeita não é bulkhead — é um gargalo com nome bonito.
- **O pool do Hikari nunca foi declarado.** O default de 10 governa o caminho de compra inteiro e não está escrito em lugar nenhum.
- **Aumentar concorrência sem medir o que vem depois** troca uma fila barata por uma cara.
- **Um limite que aperta antes esconde todos os outros.** A memória mascarou o teste de threads por completo.

---

## Pendências registradas

- Mover as duas chamadas HTTP para **fora** da `@Transactional` do `buyNow`, como o `billing` já fez.
- **Declarar o pool do Hikari** explicitamente, com valor escolhido a partir de medição e não do default.
- **Redimensionar ou repensar os `@ConcurrencyLimit`**: 10 e 15 foram escolhidos sem poder medir, e nunca chegaram a ser exercitados até esta fase. Um bulkhead com timeout de espera seria mais honesto que um que bloqueia para sempre.
- **Nenhuma métrica de pool é exposta.** `hikaricp.connections.pending` e `tomcat.threads.busy` existem no Micrometer e diriam tudo isso em produção; o Actuator aqui expõe só `health` e `info`.
- **Health check que não depende do pool da aplicação** — hoje, saturação total silencia justamente quem deveria denunciá-la.
- **Comparação faltante:** threads virtuais sob carga **dentro** da capacidade (por exemplo 400 req/s) não chegou a ser medida. O que está documentado é o comportamento na saturação. É plausível — e provável — que em regime normal as virtuais empatem ou ganhem; isso não foi verificado e não deve ser presumido.

---

## Checklist de revisão

- [ ] Antes de aumentar a concorrência, você sabe qual é o **próximo** limitador?
- [ ] O pool de conexão está declarado explicitamente?
- [ ] Existe alguma chamada de rede dentro de `@Transactional`?
- [ ] Os bulkheads rejeitam ou bloqueiam para sempre?
- [ ] O limite de memória do container foi verificado sob carga?
- [ ] `server.tomcat.threads.max` está configurado junto com threads virtuais ligadas (e portanto sem efeito)?
- [ ] Existe métrica de fila de pool exposta?

---

## Referências

- [JEP 444 — Virtual Threads](https://openjdk.org/jeps/444)
- [JEP 491 — Synchronize Virtual Threads without Pinning](https://openjdk.org/jeps/491) (Java 24+)
- [Spring Boot — Virtual threads](https://docs.spring.io/spring-boot/reference/features/task-execution-and-scheduling.html)
- [Testes de carga com k6](../03-testes-integracao/testes-de-carga-k6.md) — como os números foram obtidos
- [Resiliência](../01-arquitetura-design/resiliencia.md) — de onde vêm os bulkheads e o retry
- [Health check e degradação](./health-checks.md) — por que ninguém percebeu o travamento
