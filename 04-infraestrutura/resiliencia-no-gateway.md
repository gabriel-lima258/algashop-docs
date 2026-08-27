# Resiliência na borda: os cinco padrões chegam ao gateway — com outra biblioteca

> A [Resiliência](../01-arquitetura-design/resiliencia.md) blindou as três integrações de **saída** (ordering→catálogo, ordering→Rapidex, billing→FastPay). A borda ficou nua — e a borda multiplica: uma dependência lenta atrás do gateway pendura conexões de **todo** o tráfego, não de um cliente. Esta fase leva timeout, retry, circuit breaker, cache e um padrão novo — rate limit — para o [API Gateway](./api-gateway.md). E traz, pela primeira vez no projeto, o Resilience4j de verdade.
> Código real: `microservices/api-gateway` — `application.yaml` (`httpclient`, `default-filters`, filtros por rota, `resilience4j.circuitbreaker.instances`) e `config/security/RateLimitConfig.java`.
> Os cinco padrões em [Resiliência](../01-arquitetura-design/resiliencia.md) · a configuração dos serviços em [Resiliência na prática](./resiliencia-config.md) · o Redis compartilhado em [Redis na prática](./redis.md).

---

## Duas bibliotecas de resiliência — e qual é qual depende de onde você está

O doc de [configuração](./resiliencia-config.md) abre com uma tabela de "o que se procura × o que está no código", avisando que o projeto **não** usa Resilience4j: os serviços usam o framework-retry do Spring Framework 7, sem janela deslizante e sem limiar percentual. Esta fase inverteu a tabela — **as duas colunas agora existem, cada uma num lugar**:

| | Serviços (ordering, billing) | Gateway |
|---|---|---|
| Biblioteca | `spring-cloud-starter-circuitbreaker-framework-retry` | `spring-cloud-starter-circuitbreaker-reactor-resilience4j` |
| Stack | bloqueante (Tomcat, `RestClient`) | reativa (Netty, WebFlux) |
| Configuração | `Customizer<...>` em **Java** | `resilience4j.circuitbreaker.instances.*` em **YAML** |
| Abre por | **uma** exceção (`openTimeout`/`resetTimeout`) | **percentual de falha** numa janela (`slidingWindowSize`, `failureRateThreshold`) |

A variante importa: é `reactor-resilience4j`, não o resilience4j comum — no event loop do Netty, um breaker bloqueante travaria o gateway inteiro. E o contraste que o [doc dos cinco padrões](../01-arquitetura-design/resiliencia.md) prometeu ("é uma diferença grande em relação ao Resilience4j, cujo padrão é abrir por percentual numa janela") agora tem os dois lados rodando no mesmo sistema.

---

## Timeout: global, e é ele que habilita todo o resto

```yaml
httpclient:
  connect-timeout: 1000    # int, em MILISSEGUNDOS (contrato do Netty)
  response-timeout: 5000   # Duration sem sufixo = 5000ms — mesma armadilha do timeout: 600 do Redis
```

Sem `response-timeout`, uma chamada pendurada não é falha — é espera. É o timeout que a transforma em evento contável, e portanto é ele que permite ao Retry retentar e ao circuito abrir. A tese central do doc de resiliência, agora na borda.

O que **não** existe: override por rota (o SCG aceita `metadata: {connect-timeout, response-timeout}` por rota). O mesmo teto de 5s vale para o `GET /api/v1/products` (leitura barata, com duas camadas de cache) e para o `POST /api/v1/orders` (escrita transacional). Pendência registrada.

---

## Retry em `default-filters` é retry em TODAS as rotas

```yaml
default-filters:
  - name: Retry
    args:
      retries: 3
      statuses: BAD_GATEWAY, INTERNAL_SERVER_ERROR      # 502, 500
      methods: GET,PUT                                   # ← a linha de segurança
      exceptions: java.io.IOException, java.util.concurrent.TimeoutException
      backoff:
        firstBackoff: 10ms
        maxBackoff: 20ms
```

**`methods: GET,PUT` é a idempotência declarada em YAML.** `POST` — criar pedido, `buyNow`, o webhook do FastPay — nunca é retentado. É a mesma decisão que o billing tomou em Java para o `capture` ("retentar cobrança é cobrar duas vezes"), expressa agora como configuração de borda.

Duas assimetrias que valem leitura atenta:

- **`statuses` cobre 500 e 502, mas não 504** — e 504 está na lista do circuit breaker. Um gateway timeout do downstream não ganha nova tentativa, mas conta para abrir o circuito. Defensável (timeout provavelmente se repetiria), mas é o tipo de decisão que precisa estar escrita para não parecer esquecimento.
- **O backoff de 10–20ms é praticamente inexistente.** As 4 tentativas caem quase juntas — o downstream não ganha tempo de se recuperar. É o oposto exato da pendência do ordering (21s de backoff contra `openTimeout` de 5s): lá o retry esperava demais; aqui, de menos. E o custo real de uma dependência pendurada é a soma dos timeouts: até **4 × 5s ≈ 20s** de latência de cauda para uma única requisição GET.

### Retry por FORA do circuit breaker — e a matemática da janela

Os `default-filters` envolvem os filtros de rota. Na `ordering-route`, cada uma das 4 tentativas do Retry **atravessa o breaker e registra a própria falha**. Com `slidingWindowSize: 8` e `minimumNumberOfCalls: 5`:

> **Uma única requisição GET que falhe pode sozinha preencher 4 das 8 posições da janela. Duas requisições abrem o circuito.** O breaker parece dimensionado para 8 chamadas de clientes distintos, mas o Retry na frente muda a unidade de contagem: a janela mede *tentativas*, não *requisições*. A ordem efetiva dos filtros é conferível em `/actuator/gateway/routefilters` — e dimensionar janela sem saber quem envolve quem é chutar.

---

## Circuit breaker na rota do ordering — sem fallback, de propósito

```yaml
- id: ordering-route
  filters:
    - name: CircuitBreaker
      args:
        name: ordering-route-cb
        statusCodes: [502, 504, 500]     # ← ensina o breaker a contar RESPOSTA como falha
```

```yaml
resilience4j.circuitbreaker:
  instances:
    ordering-route-cb:
      slidingWindowType: COUNT_BASED
      slidingWindowSize: 8
      minimumNumberOfCalls: 5
      failureRateThreshold: 50
      permittedNumberOfCallsInHalfOpenState: 5
      waitDurationInOpenState: 10s
      maxWaitDurationInHalfOpenState: 30s
      registerHealthIndicator: true
```

`statusCodes` merece a pausa: do ponto de vista do transporte, um 500 do ordering é uma resposta **bem-sucedida** — chegou, completa, dentro do prazo. Sem essa lista, só exceção (timeout, conexão recusada) contaria como falha e o breaker ficaria cego para um serviço que responde erro rápido.

**Não há `fallbackUri`, e isso é escolha.** Com o circuito aberto, o gateway responde **503 cru**. O [doc de resiliência](../01-arquitetura-design/resiliencia.md) já argumentou quando um fallback mente — não dá para inventar um pedido, como não dava para inventar o preço de um produto. 503 honesto vence resposta fabricada; o dia em que existir uma degradação honesta para pedido (fila? "tente em instantes" estruturado?), o `fallbackUri` é uma linha.

`registerHealthIndicator: true` + `circuitbreakers` no Actuator: o estado do circuito aparece em `/actuator/health` e `/actuator/circuitbreakers` — a mesma observabilidade da Fase 17, agora na borda. As **outras quatro rotas não têm breaker** — billing, catálogo e authorization-server podem apodrecer sem que nada abra. Pendência.

---

## Cache local na borda: a terceira camada sobre a mesma resposta

```yaml
filter:
  local-response-cache:
    enabled: true                      # feature flag — sem ela o filtro nem registra
...
- id: product-catalog-route
  filters:
    - LocalResponseCache=5m,30MB       # timeToLive, tamanho máximo
```

O backing store é **Caffeine, in-process**: o cache vive na heap do gateway (os 30MB contam contra os 512M do container), e com N instâncias de gateway são N caches independentes — nada de Redis aqui.

A conta que importa vem do [doc de cache](../01-arquitetura-design/cache.md): **a idade de um dado é a soma das camadas.** Um produto agora pode estar: no cache server-side do catálogo (Redis db 0), no cache da borda (5min) e no cache HTTP do cliente (`max-age` de 1min que o `ProductController` emite). O `timeToLive` de 5min da borda é **maior** que o `max-age` de 1min do downstream — a borda pode servir por mais tempo do que o próprio catálogo declararia. O filtro só cacheia `GET` com 200 e respeita `no-store`; e produtos são dados públicos (`cachePublic()` do próprio catálogo) — cache compartilhado entre usuários aqui é correto *porque o dado é o mesmo para todos*, e essa premissa precisa ser reavaliada se a rota um dia servir dado por usuário.

Detalhe de ordem: o `LocalResponseCache` está declarado **antes** do `RequestRateLimiter` — um hit de cache pode nem chegar a gastar token do limite. O 50/s vira "50 misses/s", não "50 requisições/s".

---

## Rate limit: um balde por usuário, num Redis emprestado

O padrão novo do módulo — e o único que **não** protege o downstream de falha, mas de **abuso**:

```yaml
- name: RequestRateLimiter
  args:
    redis-rate-limiter:
      replenishRate: 50        # tokens repostos por segundo (taxa sustentada)
      burstCapacity: 200       # o balde: pico absorvível de uma vez
      requestedTokens: 1       # custo de cada requisição
    key-resolver: "#{@rateLimitKeyResolver}"
```

**Token bucket, atômico, no servidor.** O `RedisRateLimiter` executa um script **Lua** no Redis (db 3) — duas chaves por balde (`tokens` e `timestamp`, com TTL), decrementadas atomicamente sem round-trip de leitura-modificação-escrita. É por isso que a dependência é `data-redis-reactive`: no event loop do Netty, o client bloqueante travaria tudo. Estourou o balde → **429** com os headers `X-RateLimit-Remaining`/`-Burst-Capacity`/`-Replenish-Rate` informando o saldo. `burstCapacity` 4× o `replenishRate` significa: uma rajada de 200 é absorvida, e o balde vazio leva 4 segundos para encher de novo.

**O KeyResolver define a granularidade** — o balde não é global, é *por chave*:

```java
return exchange -> exchange.getPrincipal().map(principal -> {
        if (principal instanceof JwtAuthenticationToken jwtToken) {
            String sub = jwtToken.getToken().getClaimAsString("sub");
            return sub != null ? sub : DEFAULT_KEY;
        }
        return DEFAULT_KEY;                        // "anonymous"
    }).switchIfEmpty(Mono.just(DEFAULT_KEY))
```

Um balde de 50/s **por `sub`**: o cliente abusivo é limitado sem derrubar os demais. O `switchIfEmpty` não é enfeite — o default do filtro é `deny-empty-key: true`, e um `Mono.empty()` do resolver viraria **403** para qualquer requisição sem principal. Na prática o caminho `anonymous` quase não roda: `/api/**` exige token no gateway, então só o preflight `OPTIONS` (que é `permitAll`) cai nele — o fallback é cinto de segurança, não caminho quente.

**E o modo de falha é fail-open**: Redis fora do ar → o filtro loga e **deixa passar sem limitar**. Para rate limit é a escolha certa — indisponibilidade do limitador não pode virar indisponibilidade do produto — mas é o oposto do instinto de segurança, e por isso está escrito: quem tratar o rate limit como controle de acesso vai se surpreender.

---

## O episódio da ordem, segunda temporada

No meio deste diff, a regra `.pathMatchers("/api/**").authenticated()` foi parar **antes** do `.pathMatchers("/api/v1/webhooks/**").permitAll()`. Em `authorizeExchange`, como nas rotas, **a primeira regra que casa vence** — o `permitAll` do webhook virou código inalcançável e o FastPay (que não manda token) passaria a tomar 401: nenhuma fatura confirmaria pagamento pela borda. Corrigido nesta fase, com o comentário de ordem gravado no código.

> **A lição da Fase 34 cobrada de novo, uma camada acima:** rotas têm ordem, regras de security têm ordem, e as duas cadeias decidem de forma independente. Toda mudança em uma exige reler a outra.

---

## Armadilhas

- **`allkeys-lru` pode despejar os contadores do rate limiter.** O Redis é compartilhado (db 0 catálogo, db 1 ordering, db 3 gateway) e a política de eviction **não respeita banco lógico**: sob pressão de memória, o cache de produto pode expulsar as chaves do limitador — e chave despejada = balde cheio de novo, **limite resetado em silêncio**. Cache perdido se repõe; contador perdido mente.
- **O Redis do gateway está hardcoded — host, senha e db no YAML versionado.** Sem perfil, sem entrada no `etc/hostnames/hostnames` (não há `algashop-redis` lá) e fora do Parameter Store. Fora do Docker, o gateway não resolve o host, não conecta — e o rate limit **falha aberto sem nenhum aviso**. E se alguém trocar a senha via `.env`, tudo continua funcionando *menos* o limite.
- **`/actuator/**` inteiro está público** — incluindo `/actuator/gateway/routes` (a tabela de roteamento com service IDs) e `/actuator/circuitbreakers`. Decisão deliberada e **restrita a desenvolvimento** (é o laboratório do módulo); produção exige voltar ao health-only.
- **O 504 não é retentado, mas abre o circuito** — a assimetria `statuses` × `statusCodes` descrita acima.
- **`TRACE`/`DEBUG` de logging seguem ligados** — pendência da Fase 34, ainda de pé; o log do KeyResolver foi rebaixado para `debug` nesta fase (era uma linha com o `sub` do usuário por requisição).
- **O db 2 está vazio** — a numeração pula do 1 para o 3. Não é erro; é o tipo de detalhe que confunde quem chega depois se ninguém escrever.

## Pendências registradas

- [ ] **Redis do rate limiter sob política segura** — instância própria, ou eviction que preserve os contadores; hoje o LRU do cache pode zerar limites.
- [ ] **Config do gateway no Parameter Store** — a pendência da Fase 34 cresceu: agora são issuer, registry e **quatro valores de Redis, um deles senha**, duplicados no YAML.
- [ ] **Circuit breaker nas demais rotas** — só o ordering tem; billing, catálogo e auth-server podem cair sem nada abrir na borda.
- [ ] **`fallbackUri` quando existir degradação honesta** — hoje 503 cru, por escolha documentada.
- [ ] **Timeout por rota (`metadata`)** — 5s únicos para leituras baratas e escritas transacionais.
- [ ] **Fechar `/actuator/**` antes de produção.**
- [ ] **Testes: zero** — nem as regras de security, nem a ordem dos filtros, nem o 429 têm um teste que os perceba quebrar.
- [ ] **Confirmar a ordem efetiva dos filtros** em `/actuator/gateway/routefilters` e gravar a conclusão (Retry×breaker, cache×rate-limit).

## Checklist de revisão

- [ ] O método da rota nova é idempotente? Só então ele entra no `methods` do Retry.
- [ ] A falha nova é exceção ou resposta HTTP? Resposta só conta no breaker se estiver em `statusCodes`.
- [ ] Mexeu numa lista com ordem (rotas, security, filtros)? Releia as outras duas.
- [ ] O dado cacheado na borda é igual para todos os usuários — ainda?
- [ ] A chave do rate limit tem a granularidade da ameaça (usuário? IP? client)?
- [ ] O que acontece quando o Redis do limitador cai — e todo mundo sabe que é fail-open?
- [ ] Janela do breaker dimensionada em requisições ou em tentativas? O Retry na frente muda a unidade.

## Referências

- [Spring Cloud Gateway — GatewayFilter Factories (Retry, CircuitBreaker, RequestRateLimiter, LocalResponseCache)](https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway-server-webflux/gatewayfilter-factories.html)
- [Resilience4j — CircuitBreaker](https://resilience4j.readme.io/docs/circuitbreaker)
- [Stripe — Scaling your API with rate limiters](https://stripe.com/blog/rate-limiters)
- [Resiliência](../01-arquitetura-design/resiliencia.md) · [Resiliência na prática](./resiliencia-config.md) · [API Gateway](./api-gateway.md) · [Redis na prática](./redis.md) · [Cache](../01-arquitetura-design/cache.md)
