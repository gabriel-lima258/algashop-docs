# Health check, readiness e degradação

> "Está no ar" e "consegue trabalhar" são perguntas diferentes. Como o Actuator responde as duas, por que existe um status `DEGRADED` inventado por este projeto — e o que a verificação mostrou que funciona e o que não funciona.
> Código real: `CustomFrameworkRetryCircuitBreakerHealthIndicator.java` e `CustomRedisCacheHealthIndicator.java` (ordering, billing e catalog), `application-base.y*ml` dos três serviços.

> Este documento fecha uma pendência da Fase 16: *"Nenhuma observabilidade dos circuitos. Sem Actuator, sem métrica de abertura."*

---

## O problema

Um orquestrador precisa decidir duas coisas sobre um container, e elas não são a mesma:

| Pergunta | Se a resposta for não |
|---|---|
| **Liveness** — o processo está vivo? | reiniciar |
| **Readiness** — ele consegue atender? | tirar de rotação, sem reiniciar |

Misturar as duas custa caro nos dois sentidos. Um serviço que responde "não estou vivo" porque o banco caiu vai ser **reiniciado em loop** — e reiniciar não conserta banco de dados. Um serviço que responde "estou pronto" porque o processo subiu vai **receber tráfego que não consegue atender**.

E existe um terceiro caso, que é o mais interessante e o que a maioria das implementações erra:

> **Dependência opcional fora do ar.** O Redis caiu. O serviço continua respondendo tudo — mais devagar, indo ao banco. Ele não deve ser reiniciado nem tirado de rotação. Mas também não está *bem*, e alguém precisa saber.

Nenhum dos dois estados padrão serve: `UP` esconde o problema, `DOWN` causa um estrago maior que o problema.

---

## `DEGRADED` — um status inventado

```yaml
management:
  endpoint:
    health:
      status:
        order: "DOWN, OUT_OF_SERVICE, UNKNOWN, DEGRADED, UP"
```

`DEGRADED` **não existe no Spring**. É uma string arbitrária, e funciona porque `Health.status(String)` aceita qualquer código — `Status` não é um enum, e `UP`/`DOWN` são apenas constantes de conveniência.

Quem dá significado ao código é o `status.order`: a lista vai do **mais severo ao menos severo**, e o status agregado é o pior encontrado. Colocar `DEGRADED` entre `UNKNOWN` e `UP` diz exatamente: *pior que saudável, melhor que desconhecido, e bem longe de fora do ar*.

O vocabulário completo do projeto:

| Situação | Status | Consequência pretendida |
|---|---|---|
| Banco fora | `DOWN` | não receber tráfego |
| Redis fora | `DEGRADED` | serve mais devagar, mas serve |
| Circuito aberto | `DEGRADED` | a integração caiu, o resto funciona |

> ⚠️ **E aqui está a limitação que torna `DEGRADED` quase decorativo hoje: ele devolve HTTP 200.**
>
> O mapeamento padrão do Actuator manda para 503 apenas `DOWN` e `OUT_OF_SERVICE`. Qualquer código desconhecido — e `DEGRADED` é desconhecido para ele — cai no 200. Verificado:
>
> ```
> $ curl -o /dev/null -w "%{http_code}" localhost:8081/actuator/health
> 200          # com o circuito ABERTO e o agregado DEGRADED
> ```
>
> Ou seja: **um probe que olhe o código de status não vê diferença nenhuma.** A distinção só existe para quem lê o corpo da resposta. Fechar isso pediria `management.endpoint.health.status.http-mapping`.

---

## O grupo `readiness` — a decisão de desenho

```yaml
group:
  readiness:
    include: db,readinessState        # ordering e billing
    include: mongo,readinessState     # product-catalog
```

Só o banco entra. **Nem cache nem circuitos.**

É a mesma decisão de antes, dita do outro lado: `/actuator/health/readiness` responde *"posso receber requisição?"*, não *"está tudo bem?"*. Tirar a instância do balanceador porque um terceiro caiu transformaria uma falha parcial em indisponibilidade total — exatamente o que a resiliência da Fase 16 existe para evitar.

Verificado com o circuito da Rapidex aberto:

```
GET /actuator/health              →  status: DEGRADED
GET /actuator/health/readiness    →  status: UP    ← intocado
```

O grupo faz o que promete.

> Sobre o `readinessState`: ele aparece no grupo e **está registrado**, apesar de `management.endpoint.health.probes.enabled` não estar definido em nenhum perfil. No Boot 4 os indicadores de disponibilidade sobem sem exigir detecção de Kubernetes. Confirmado no endpoint — `livenessState` e `readinessState` aparecem entre os componentes.

---

## Os dois indicadores customizados

### Circuit breakers — funciona, e foi provado

```java
@Component("circuitbreakers")
public class CustomFrameworkRetryCircuitBreakerHealthIndicator implements HealthIndicator {
    // CLOSED / HALF_OPEN -> UP        OPEN -> DEGRADED
}
```

O construtor pede os circuitos à factory **pelo mesmo id que os clients usam** — daí os ids terem virado constantes em `SpringCircuitBreakerConfig`. Isso só funciona se `create(id)` devolver a instância já configurada, e não uma nova: o estado vive dentro do `CircuitBreakerRetryPolicy`, então uma instância nova reportaria `CLOSED` para sempre, com o endpoint respondendo bonito e mentindo.

**Provado que devolve a mesma.** Forçando a Rapidex a responder 500:

```
antes:                                    depois da falha:
"circuitbreakers": {                      "circuitbreakers": {
  "status": "UP",                           "status": "DEGRADED",
  "details": {                              "details": {
    "rapidexAPICB":    {"state":"CLOSED"},    "rapidexAPICB": {
    "productCatalogCB":{"state":"CLOSED"}       "state": "OPEN",
  }                                             "lastException": "Rapidex API Bad Gateway"
}                                             },
                                              "productCatalogCB": {"state":"CLOSED"}
                                            }
                                          }
```

Três coisas de uma vez: o estado real chega ao endpoint; o agregado vai a `DEGRADED` e **não** a `DOWN`, provando o `status.order`; e `productCatalogCB` segue `CLOSED`, provando que os circuitos são independentes.

### Cache — **não funciona**

```java
@Component("cache")
@ConditionalOnProperty(name = "spring.cache.type", havingValue = "redis")
public class CustomRedisCacheHealthIndicator implements HealthIndicator {
    public Health health() {
        try {
            redisConnectionFactory.getConnection().ping();
            return Health.up().build();
        } catch (Exception e) {
            return Health.status("DEGRADED")...
        }
    }
}
```

O desenho está certo: desligar o indicador nativo (`management.health.redis.enabled: false`, que reportaria `DOWN`) e pôr no lugar um que reporte `DEGRADED`. As duas peças só funcionam juntas — sem desligar o nativo, seriam dois indicadores e o dele venceria.

**Mas ele reporta `UP` sem falar com o Redis.** Três medições independentes:

```
1. Redis PARADO (docker stop), três leituras seguidas:
   cache = UP, UP, UP

2. Dois GET /actuator/health, contando comandos no servidor:
   CONFIG RESETSTAT → 2 chamadas → cmdstat_ping do app: 0

3. CLIENT LIST no Redis, com o serviço no ar:
   1 cliente — o próprio redis-cli. A aplicação: nenhuma conexão.
```

O `ping()` retorna sem exceção e sem chegar ao servidor. Um indicador que responde `UP` sem verificar nada é **pior que não ter indicador**: ele produz confiança onde não há informação.

> **Isto é a mesma falha da Fase 15**, vista por outro ângulo. Lá o sintoma foi o cache nunca populando, com `DBSIZE` em zero e nenhuma conexão aberta; aqui é um `ping()` explícito que também não abre conexão. A causa raiz segue **não identificada**, e agora há duas evidências apontando para o mesmo lugar: nesta aplicação o cliente Redis não estabelece conexão, e falha em silêncio nas duas vezes. Ver [`cache.md`](../01-arquitetura-design/cache.md).

---

## A armadilha do discovery client

```yaml
spring:
  cloud:
    discovery:
      client:
        health-indicator:
          enabled: false
```

A linha mais sutil de toda a configuração, e ela existe por um efeito colateral em cadeia:

1. O starter de circuit breaker traz o `spring-cloud-commons` transitivamente
2. O `spring-cloud-commons` registra um `DiscoveryClientHealthIndicator`
3. Não há Eureka nem Consul aqui — as chamadas usam URL fixa — então ele fica **`UNKNOWN` para sempre**
4. E `UNKNOWN` está acima de `UP` no `status.order`

Resultado sem essa linha: o `/actuator/health` **nunca sairia de `UNKNOWN`**, e pararia de distinguir serviço são de serviço com problema. Um indicador irrelevante envenenando o agregado inteiro.

> A lição é sobre o `status.order`: colocar `UNKNOWN` acima de `UP` é razoável — "não sei" é pior que "sei que está bem". Mas isso torna **qualquer** indicador mal comportado capaz de derrubar o agregado. Quem escolhe essa ordem assume o trabalho de auditar o que entra no classpath.

---

## O nome do bean é o contrato

```java
@Component("cache")            →  /actuator/health  ...  "components": { "cache": ... }
@Component("circuitbreakers")  →                          "circuitbreakers": ...
```

Sem o nome explícito, o Boot derivaria do nome da classe: `customRedisCache` e `customFrameworkRetryCircuitBreaker`. Detalhe pequeno com consequência grande — **renomear a classe mudaria o corpo do `/actuator/health`**, que é consumido por probes e painéis. O nome do bean aqui é API, não detalhe interno.

---

## Armadilhas

1. **`DEGRADED` devolve HTTP 200.** Só `DOWN` e `OUT_OF_SERVICE` viram 503 por padrão.
2. **Indicador que não verifica nada reporta UP.** Foi o que aconteceu com o cache.
3. **Dependência transitiva pode registrar indicador que envenena o agregado.**
4. **`UNKNOWN` acima de `UP` no `status.order`** transforma todo indicador desconhecido em problema.
5. **O nome do bean é o nome do componente no endpoint** — renomear a classe quebra quem consome.
6. **Readiness com dependência opcional dentro** tira a instância de rotação por motivo errado.
7. **Reiniciar não conserta dependência externa** — por isso liveness não deve olhar para fora.

---

## Pendências registradas

- [ ] **O indicador de cache não detecta o Redis fora do ar.** Reporta `UP` sem abrir conexão — provado por `CLIENT LIST` e por `commandstats`. Mesma família do problema aberto na Fase 15.
- [ ] **O ciclo do health check está aberto.** Os três serviços expõem `/actuator/health/readiness` e **nada o consome**: nenhum `HEALTHCHECK` nos Dockerfiles, nenhum `healthcheck:` nos serviços de aplicação do compose. Na prática o Docker considera o container saudável assim que o processo sobe. Fechar exigiria `curl`/`wget` na imagem — a base `eclipse-temurin:25-jre` não tem nenhum dos dois — ou um bloco no compose.
- [ ] **`DEGRADED` sem mapeamento HTTP.** Enquanto devolver 200, só serve para leitura humana.
- [ ] **`show-details: always` sem Spring Security.** Não há autenticação em nenhum dos três serviços, e `management.info.env.enabled: true` acompanha. Qualquer um que alcance a porta vê estado dos circuitos, a última exceção do gateway e o ambiente. Aceitável localmente; num ambiente real seriam `management.server.port` separado e Security.
- [ ] **`DEGRADED` é string literal repetida.** Três ocorrências por serviço — duas no Java (uma comparada com `.equals`) e uma no YAML. Mudar num lugar e não no outro tira o status da ordenação em silêncio.
- [ ] **Os indicadores são duplicados entre serviços**, com diferença de uma a três linhas. Um módulo compartilhado resolveria; não existe um.
- [ ] **Ids de circuito hardcoded no indicador.** Um circuito novo não aparece sem editar a classe.
- [ ] **`fastpayReadCB` vale `"fastpayCB"`** — nome e valor divergem. E o `ordering` usa sufixo `Id` nas constantes, o `billing` não.
- [ ] **`billing-scheduler` não tem health, e não deve ter.** É um job efêmero sem porta HTTP: não há readiness a responder. Fica registrado para não parecer esquecimento.

---

## Checklist de revisão

- [ ] Sei dizer a diferença entre liveness e readiness, e o que cada resposta negativa provoca
- [ ] Entendo por que existe um terceiro caso que nenhum dos dois cobre
- [ ] Sei como um status novo entra no Spring sem nenhuma classe
- [ ] Sei por que a ordem em `status.order` é o que dá significado ao `DEGRADED`
- [ ] Sei que `DEGRADED` devolve 200, e o que isso invalida
- [ ] Entendo por que o indicador nativo do Redis precisou ser desligado
- [ ] Sei por que o grupo readiness não inclui cache nem circuitos
- [ ] Entendo como uma dependência transitiva pode envenenar o agregado
- [ ] Sei por que o nome do bean do indicador é contrato
- [ ] Sei reconhecer um indicador que reporta UP sem verificar nada

---

## Referências

- [Spring Boot — Actuator Health](https://docs.spring.io/spring-boot/reference/actuator/endpoints.html#actuator.endpoints.health)
- [Spring Boot — Kubernetes Probes](https://docs.spring.io/spring-boot/reference/actuator/endpoints.html#actuator.endpoints.kubernetes-probes)
- [Kubernetes — Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/)
- [`resiliencia.md`](../01-arquitetura-design/resiliencia.md) — os circuitos que este endpoint observa
- [`resiliencia-config.md`](./resiliencia-config.md) — os parâmetros dos circuitos
- [`cache.md`](../01-arquitetura-design/cache.md) — o cache que o indicador deveria observar, e não observa
- [`ambiente-local.md`](./ambiente-local.md) — portas e como subir tudo
