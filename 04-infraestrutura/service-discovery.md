# Service discovery: o endereço saiu da configuração

> O módulo de segredos centralizados tirou as URLs do YAML e as pôs no Parameter Store — mas cada uma continuava sendo **um** endereço, apontando para **uma** instância. Este módulo troca o endereço pelo nome: um Eureka Server vira o registro central, quatro serviços se registram nele, e a chamada `ordering → product-catalog` passa a resolver o destino em runtime, com balanceamento entre as instâncias que existirem.
> Código real: `microservices/service-registry` (novo — `@EnableEurekaServer`) · `EurekaClientConfig` + bloco `eureka:` nos 4 clients · `RestClientBuilderConfig` e `ProductCatalogApiConfig` (ordering) · `docker-compose.services.yml` · `etc/aws/parameters.csv`.
> A configuração centralizada em [Segredos centralizados](../05-seguranca/segredos-centralizados-e-chave-rsa.md) · a chamada que foi balanceada em [Resiliência na prática](./resiliencia-config.md) · o indicador de health que esta fase ressignifica em [Health checks](./health-checks.md).

---

## O problema: endereço fixo escala até 1

`http://product-catalog` no lugar de `http://localhost:8083` parece uma troca cosmética. Não é — é a diferença entre *saber onde o serviço está* e *perguntar a quem sabe*.

Com endereço fixo, três coisas são verdade ao mesmo tempo:

1. **Uma instância por serviço.** Se o catálogo ganhar uma réplica, quem distribui as chamadas? A URL aponta para uma máquina.
2. **O endereço é configuração.** Mudar a porta, o host, a topologia — tudo exige editar parâmetro e reiniciar quem chama.
3. **Ninguém sabe o que está de pé.** A lista de serviços vivos não existe em lugar nenhum; cada chamador descobre na base do erro de conexão.

O *service registry* inverte as três: cada serviço **anuncia a si mesmo** ao subir (nome, IP, porta, saúde), o registro mantém a lista viva, e quem chama pergunta pelo **nome** — recebendo a lista de instâncias e escolhendo uma. Endereço deixa de ser configuração e vira **estado de runtime**.

---

## O registro: um servidor que precisa se desligar de si mesmo

O `service-registry` é o sexto repositório de serviço do projeto — e o menor: uma classe, um YAML, um Dockerfile.

```java
@SpringBootApplication
@EnableEurekaServer
public class ServiceRegistryApplication { ... }
```

O detalhe que rende a explicação está no `application.yaml`:

```yaml
eureka:
  client:
    registerWithEureka: false
    fetchRegistry: false
```

**Toda aplicação Spring Cloud Netflix embute um Eureka *client*** — inclusive o servidor. Sem essas duas linhas, o registro tentaria registrar-se em si mesmo e sincronizar a lista de serviços com um *peer* que não existe, logando erro em loop. As duas flags só voltam a ser `true` num cluster de Eureka Servers replicados, onde cada nó é cliente dos outros — aqui é uma instância standalone.

O dashboard fica em `http://localhost:8761` (a porta padrão do Eureka, publicada 1:1 no compose) e mostra quem está registrado, com qual `instanceId` e em que estado.

> Nota de estrutura: o registry **não** segue o esquema de perfis dos outros serviços (`base + development-env + docker-env`). Tudo mora no `application.yaml` raiz, e o `SPRING_PROFILES_ACTIVE: docker` que o compose injeta cai num perfil vazio — funciona, mas por ausência de conteúdo, não por desenho. Se um dia o registry precisar de config por ambiente, o esquema dos vizinhos está a um copy-paste de distância.

---

## Os clients: quatro serviços que se anunciam

`ordering`, `billing`, `product-catalog` e `authorization-server` ganharam o mesmo trio: a dependência (`spring-cloud-starter-netflix-eureka-client`), uma config de marcação —

```java
@Configuration
@EnableDiscoveryClient
public class EurekaClientConfig {
}
```

— e o bloco no `application-development-env`:

```yaml
eureka:
  client:
    serviceUrl:
      defaultZone: ${shared.params.service-registry-url}/eureka
    healthcheck:
      enabled: true
  instance:
    instanceId: ${random.uuid}
    prefer-ip-address: true
```

Quatro decisões, uma por linha:

**`${shared.params.service-registry-url}`** — a URL do registry é o **segundo parâmetro do namespace `shared`**, depois do issuer. O mesmo padrão do módulo anterior: o endereço que todo mundo precisa vive uma vez no Parameter Store (`http://algashop-service-registry:8761`), e o `hosts` da máquina ganhou a linha que faz esse nome resolver fora do Docker — o truque já usado para o LocalStack e o Postgres.

**`healthcheck.enabled: true`** — por padrão o Eureka considera a instância `UP` enquanto o *heartbeat* chegar, mesmo que a aplicação esteja com o banco fora. Com a flag, o client propaga o estado do `/actuator/health` real para o registro — um serviço `DOWN` some da lista antes de morrer de verdade. É por causa desse acoplamento com o Actuator que as quatro `SecurityConfig` liberaram `/actuator/info/**` junto do health.

**`instanceId: ${random.uuid}`** — o id default combina hostname e porta; duas réplicas no mesmo host colidiriam. Com UUID por subida, **cada réplica é uma entrada distinta** no registro — a premissa do balanceamento.

**`prefer-ip-address: true`** — na rede do compose, o hostname interno do container não é necessariamente resolvível por quem consulta o registro; o IP é. O client anuncia o IP em vez do nome.

E os três `logging.level ... OFF` do Netflix em dev existem porque o Eureka client é *falador* — reconexão, heartbeat, cache — mas têm custo: uma reconexão silenciosa também é uma **des**conexão silenciosa. Registrado nas armadilhas.

### Quem ficou de fora — e por quê

| Quem | Registra? | Motivo |
|---|---|---|
| `billing-scheduler` | não | Job efêmero, sem porta HTTP: sobe, executa e encerra com código 0. Registrar/desregistrar a cada execução só poluiria o registro. |
| Rapidex | não | Integração **externa** — não há instância nossa para registrar; a URL fixa continua sendo o modelo certo. |
| `authorization-server` | **sim, mas ninguém o descobre por lá** | O issuer é, por natureza, uma URL fixa (`iss` do token é comparado literalmente pelos resource servers). O registro serve hoje à visibilidade no dashboard, não ao roteamento. |

---

## O balanceador: dois builders, e o porquê de serem dois

A parte com mais sutileza do módulo está num arquivo de 20 linhas do `ordering`:

```java
@Configuration
public class RestClientBuilderConfig {

    @Bean
    @Primary
    public RestClient.Builder restClientBuilder(RestClientBuilderConfigurer configurer) {
        return configurer.configure(RestClient.builder());
    }

    @Bean
    @LoadBalanced
    public RestClient.Builder loadBalancedRestClientBuilder(RestClientBuilderConfigurer configurer) {
        return configurer.configure(RestClient.builder());
    }
}
```

**Por que dois?** Ao declarar um `RestClient.Builder` anotado com `@LoadBalanced`, o builder auto-configurado do Boot deixa de ser o único candidato — e todo ponto de injeção do serviço precisa decidir qual quer. O `ordering` tem **dois públicos de chamada**: o catálogo, que está no registry e deve ser balanceado, e o Rapidex, que é externo e **não pode** passar pelo balanceador (o LB trataria `localhost` como service ID e não acharia nada). Daí o par: o `@Primary` cru atende quem não pediu nada — o Rapidex continua funcionando sem tocar em uma linha — e o `@LoadBalanced` atende quem pedir explicitamente:

```java
@Bean
public ProductCatalogApiClient productCatalogApiClient(
        @LoadBalanced RestClient.Builder builder, ...) {
```

Os dois passam pelo `RestClientBuilderConfigurer` — é ele que reaplica as customizações que a autoconfiguração do Boot faria, para que "declarei meu próprio builder" não signifique "perdi os defaults".

**O que o balanceador intercepta.** A URL configurada virou `http://product-catalog` — repare: o esquema continua `http://`, não existe `lb://` aqui (isso é sintaxe de Spring Cloud Gateway). O que muda é que o *host* da URL é tratado como **service ID**: um interceptor no builder pergunta ao Eureka pelas instâncias de `product-catalog`, escolhe uma (round-robin por padrão) e reescreve o host pelo IP:porta real antes de a requisição sair.

**E o que não mudou — que é o ponto.** O corpo do `ProductCatalogApiConfig` é o mesmo: `baseUrl` das properties, timeouts de 3s/7s, interceptor OAuth2, `HttpServiceProxyFactory` gerando o client declarativo de `@GetExchange`. A camada de porta hexagonal não percebeu o discovery — ele entrou por baixo do builder, exatamente onde a infraestrutura deve morar.

### O vocabulário de falha mudou junto

Antes, tudo que a chamada podia lançar era `RestClientException` — e o `ResilientProductCatalogAPIClient` traduzia isso para 502/504 dentro do circuito. Com o balanceador na frente, apareceu um modo de falha novo: **nenhuma instância registrada**. Esse erro nasce no LB, *antes* de existir requisição HTTP, e não é `RestClientException`. O `catch` foi alargado:

```java
} catch (Exception e) {          // era RestClientException
    log.warn("Product Catalog call failed | {}", describe(e));
    throw translateException(e);
}
```

Sem isso, o "registry vazio" passaria cru pelo tratamento e chegaria ao usuário como 500 anônimo, invisível para retry e circuit breaker. O preço está registrado: `Exception` é um martelo grande — um `NullPointerException` do próprio código agora também vira "Bad Gateway". O refinamento (capturar `IllegalStateException`/`NoInstancesAvailableException` além de `RestClientException`) fica como pendência.

> **Discovery não é só um jeito novo de achar o endereço — é um jeito novo de falhar.** Quem põe um balanceador na frente de um client resiliente precisa reapresentar os erros novos ao tratamento antigo.

---

## O indicador que voltou a ter sentido

Desde a fase de health checks, `ordering` e `billing` carregam:

```yaml
spring.cloud.discovery.client.health-indicator.enabled: false
```

O comentário da época explicava: *"não há Eureka nem Consul aqui — as chamadas usam URL fixa — então ele fica `UNKNOWN` para sempre"*. A premissa **caiu nesta fase**: agora há Eureka, e o indicador reportaria estado real do discovery (conectado, lista de serviços). A linha que antes removia ruído passou a esconder sinal.

Não foi reativada de propósito — ligar o indicador o coloca no agregado de health, e é preciso conferir o efeito no `readiness` antes (um registry fora do ar não deveria tirar o serviço de rotação: o discovery aqui é dependência da *chamada ao catálogo*, que já tem fallback próprio). Pendência registrada, dos dois lados — aqui e em [Health checks](./health-checks.md).

---

## Armadilhas

- **Override de perfil vence o Parameter Store — e reintroduziu o endereço fixo.** O `application-docker-env.yaml` do ordering ainda tinha `product-catalog.url: http://algashop-product-catalog:8083`. Com o builder balanceado, esse host é interpretado como **service ID** — e o catálogo se registra como `product-catalog`, não como o nome do container. O perfil docker quebraria a chamada. Corrigido para `http://product-catalog`; a lição é que centralizar a config não elimina os overrides — e cada override é um lugar onde a migração pode ficar pela metade.
- **`@LoadBalanced` muda a semântica do host para TODO client que usar aquele builder.** URL de serviço externo num builder balanceado não "passa direto" — falha procurando um service ID que não existe. Por isso os dois builders, e por isso o Rapidex fica no cru.
- **O perfil `production` não tem bloco `eureka`** — como o `config.import` da fase anterior, a configuração vive só em `development-env` (que o grupo `docker` herda). Production sobe sem discovery.
- **`restart: no` num serviço de infraestrutura** — o registry herdou o padrão dos jobs. Se ele morrer, nada o levanta, e os clients seguem operando com o cache local do registro até ele voltar... manualmente.
- **Os `logging OFF` do Netflix** silenciam também as reconexões — o serviço pode passar minutos sem registro e o log de dev não conta.
- **O healthcheck do compose é teste de porta, não de readiness** — a imagem `temurin-jre` não traz `curl`, então o check usa `bash /dev/tcp`. Suficiente para ordenar a subida; não confundir com o health do Actuator.

## Pendências registradas

- [ ] **Refinar o `catch (Exception e)`** do client do catálogo para os tipos reais do LB, devolvendo o `NullPointerException` ao dono legítimo (o bug).
- [ ] **Reativar `discovery.client.health-indicator`** avaliando o impacto no readiness — hoje a linha esconde o sinal que passou a existir.
- [ ] **`production-env` sem Eureka** nos quatro clients — mesma família da pendência de config da fase anterior.
- [ ] **O dashboard 8761 está aberto** — sem autenticação, qualquer um na rede lista os serviços, IPs e portas.
- [ ] **Registry standalone** — sem peer, o registro é ponto único de falha (mitigado pelo cache local dos clients); cluster de Eureka fica para quando houver mais de uma máquina de verdade.
- [ ] **`billing-scheduler` fora do discovery é decisão, não esquecimento** — registrada aqui para não virar "falta" numa leitura futura.

## Checklist de revisão

- [ ] O serviço novo se registra? Então: starter, `@EnableDiscoveryClient`, bloco `eureka:` com `defaultZone` do `shared`, `instanceId` aleatório.
- [ ] A chamada nova é para serviço interno ou externo? Interno → builder `@LoadBalanced` + service ID na URL; externo → builder `@Primary` + URL fixa.
- [ ] O service ID da URL bate com o `spring.application.name` de quem atende?
- [ ] Algum override de perfil (docker/production) ainda carrega endereço fixo da rota migrada?
- [ ] O tratamento de erro do client enxerga as falhas do balanceador, além das de HTTP?
- [ ] O compose declara `depends_on` do registry para o client novo?
- [ ] O dashboard mostra a instância com o estado esperado depois do `healthcheck.enabled`?

## Referências

- [Spring Cloud Netflix — Eureka Server e Client](https://docs.spring.io/spring-cloud-netflix/reference/spring-cloud-netflix.html)
- [Spring Cloud LoadBalancer](https://docs.spring.io/spring-cloud-commons/reference/spring-cloud-commons/loadbalancer.html)
- [Microservices.io — Service registry / Client-side discovery](https://microservices.io/patterns/service-registry.html)
- [Segredos centralizados e a chave RSA](../05-seguranca/segredos-centralizados-e-chave-rsa.md) · [Health checks](./health-checks.md) · [Resiliência na prática](./resiliencia-config.md) · [Ambiente local](./ambiente-local.md)
