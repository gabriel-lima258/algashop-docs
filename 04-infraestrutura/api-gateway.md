# API Gateway: a porta de entrada virou uma só

> O [service discovery](./service-discovery.md) tirou o endereço da configuração para as chamadas *entre* serviços. Faltava a outra direção: quem vem de fora ainda conhecia quatro portas, e cada serviço barrava anônimo e negociava CORS por conta própria. Esta fase põe um Spring Cloud Gateway na frente de tudo — porta 9999, rotas resolvidas no Eureka, CORS num lugar só e token validado antes de a requisição tocar qualquer serviço.
> Código real: `microservices/api-gateway` (novo) — `application.yaml` (rotas, globalcors), `GatewayEcommerceSecurityConfig` (resource server reativo), `EurekaClientConfig` · `docker-compose.services.yml`.
> A resolução por service ID em [Service discovery](./service-discovery.md) · quem valida o quê em [Resource servers e escopos](../05-seguranca/resource-server-e-escopos.md) · o acoplamento de subida que voltou em [OAuth2 client e token](../05-seguranca/oauth2-client-e-token.md).

**Aviso de vocabulário:** neste projeto "gateway" já era o FastPay — *gateway de pagamento*. São coisas sem parentesco: o FastPay é um provedor externo que o billing chama; o API gateway é a porta de entrada HTTP do sistema. Este documento é sobre o segundo.

---

## O problema: N portas, N configurações de borda

Antes desta fase, "chamar o AlgaShop" significava saber que pedido é `:8081`, fatura é `:8082`, produto é `:8083` e usuário é `:9000`. Três consequências, todas na borda:

1. **O front conhece a topologia.** Cada serviço novo, cada porta movida, é mudança no cliente.
2. **CORS em N lugares** — ou, na prática, em lugar nenhum: os serviços de negócio tinham `cors.disable()` e contavam com ninguém chamar de navegador.
3. **Anônimo viaja até o destino.** Uma requisição sem token atravessa a rede toda para ser barrada pelo resource server do serviço final.

O gateway inverte os três: **uma** porta (9999), **um** ponto de CORS, e o 401 acontece na borda — antes de existir roteamento.

---

## Rotas à mão — e a ordem é regra de negócio

O Spring Cloud Gateway tem um modo automático (`discovery.locator.enabled`) que criaria uma rota `/SERVICE-ID/**` para **tudo** que estiver no registry — inclusive o próprio registry. Ficou de fora de propósito: rota declarada à mão é superfície de exposição declarada à mão.

```yaml
routes:
  - id: billing-route
    uri: lb://billing
    predicates:
      - Path=/api/v1/customers/me/orders/{orderId}/invoice,/api/v1/orders/{orderId}/invoice,/api/v1/customers/me/credit-cards/**
  - id: billing-webhook-route
    uri: lb://billing
    predicates:
      - Path=/api/v1/webhooks/fastpay
      - Method=POST
  - id: ordering-route
    uri: lb://ordering
    predicates:
      - Path=/api/v1/orders/**,/api/v1/customers/**,/api/v1/shipping-cost-previews
  - id: product-catalog-route
    uri: lb://product-catalog
    predicates:
      - Path=/api/v1/products/**,/api/v1/categories/**,/api/v1/upload-requests
  - id: user-management-route
    uri: lb://authorization-server
    predicates:
      - Path=/api/v1/users/**
```

Dois detalhes sustentam o arquivo:

**`lb://billing` é service ID, não endereço.** O host depois do `lb://` é o `spring.application.name` registrado no Eureka — o `ReactiveLoadBalancerClientFilter` resolve as instâncias e balanceia. É o mesmo mecanismo do [`@LoadBalanced` do ordering](./service-discovery.md), agora na borda: o gateway não conhece porta de ninguém.

**A ordem das rotas decide quem atende.** `billing` e `ordering` dividem o prefixo `/api/v1/customers/**`: o carrinho e o perfil são do ordering, mas `/api/v1/customers/me/credit-cards` e `/api/v1/customers/me/orders/{id}/invoice` são do billing. As rotas são avaliadas **na ordem do arquivo** — o billing, mais específico, vem primeiro; invertê-las mandaria toda fatura e cartão para o ordering, que responderia 404. A ordem aqui não é estética: é a tabela de roteamento do domínio, e o comentário no YAML existe para ninguém "organizar" o arquivo alfabeticamente.

### O gateway consulta o registry e não se anuncia

```yaml
eureka:
  client:
    registerWithEureka: false   # ninguém descobre o gateway - ele é a borda
    fetchRegistry: true         # mas ele descobre todo mundo
```

É o espelho invertido do próprio Eureka Server ([que registra sem consultar ninguém](./service-discovery.md)): o gateway consome o registro sem entrar nele. Nenhum serviço interno tem por que resolver "api-gateway" — quem precisa do gateway está do lado de fora.

---

## Token na borda, autorização no destino

O gateway **é um resource server** — o quinto do sistema, e o primeiro reativo (WebFlux, `SecurityWebFilterChain` em vez de `SecurityFilterChain`):

```java
http.cors(Customizer.withDefaults())
    .csrf(ServerHttpSecurity.CsrfSpec::disable)
    .authorizeExchange(authorize -> authorize
            .pathMatchers("/actuator/health").permitAll()
            .pathMatchers(HttpMethod.OPTIONS, "/api/**").permitAll()   // preflight
            .pathMatchers("/api/v1/webhooks/**").permitAll()           // FastPay não tem token
            .pathMatchers("/api/**").authenticated()
            .anyExchange().denyAll()
    )
    .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()));
```

O que a borda verifica: assinatura (via `/oauth2/jwks` do issuer), `iss`, `exp`. O que a borda **não** verifica: escopo, papel, dono do recurso. E isso é desenho, não lacuna:

> **O gateway é a primeira barreira, não a única.** O header `Authorization` é repassado ao serviço de destino (o Spring Cloud Gateway, ao contrário do Zuul, não o trata como sensível), e lá o mesmo token é validado **de novo** — agora com a [matriz completa](../05-seguranca/resource-server-e-escopos.md): escopo por rota, papel por público, dono na consulta. O token é validado duas vezes de propósito: se alguém alcançar um serviço por dentro da rede, sem passar pela borda, a segurança continua inteira. Ler "agora a segurança está no gateway" seria desfazer três fases de trabalho.

### `denyAll` como default — e a rota que morreu por causa dele

Tudo que não é `/api/**` nem `/actuator/health` responde 403. Esse default fechado teve uma vítima ilustrativa: existia uma rota de estudo `Path=/products/**` com `RewritePath` reescrevendo para `/api/v1/products/...` — e ela **nunca funcionou**, porque `/products/**` não casa com nenhum `pathMatchers` permitido e o 403 acontece **antes de o roteamento rodar**. A rota foi removida nesta fase; o filtro fica registrado como exemplo:

```yaml
filters:
  - RewritePath=/products(?<segment>/?.*), /api/v1/products$\{segment}
```

> **Security e roteamento são cadeias separadas — e a security decide primeiro.** Toda rota nova precisa de duas perguntas: o predicate casa? e o `authorizeExchange` deixa chegar lá? Esquecer a segunda produz exatamente o 403 que parece bug de rota e é decisão de segurança.

### O webhook público, roteado

`/api/v1/webhooks/fastpay` é `permitAll` no gateway **e** no billing — a exceção pública documentada desde a matriz do billing agora existe em duas camadas coerentes. A ironia pendente: o `webhook-url` configurado no FastPay ainda aponta **direto** para o billing (`host.docker.internal:8082`) — a rota existe e ninguém a usa. Registrado nas pendências.

---

## CORS num lugar só (quase)

```yaml
globalcors:
  add-to-simple-url-handler-mapping: true
  cors-configurations:
    '[/**]':
      allowedOrigins: http://admin.algashop.local:4200
      allowedHeaders: '*'
      allowedMethods: [GET, POST, PUT, DELETE, OPTIONS]
default-filters:
  - DedupeResponseHeader=Access-Control-Allow-Origin Access-Control-Allow-Credentials, RETAIN_FIRST
```

O navegador negocia CORS com **quem ele chama** — e agora ele chama o gateway. Os serviços de negócio podem continuar com `cors.disable()`, que sempre tiveram: nenhum navegador fala com eles diretamente. O `DedupeResponseHeader` com `RETAIN_FIRST` é a guarda para o caso de um downstream mandar os próprios headers de CORS: prevalecem os do gateway, e o navegador não vê o header duplicado que o faria rejeitar a resposta.

A exceção que confirma a regra: o **authorization server mantém o `CorsConfig` próprio** — o SPA fala com ele *diretamente* (login, PKCE, silent refresh via iframe), num fluxo que não passa e não deve passar pelo gateway. CORS mora onde o navegador bate; agora são dois lugares, cada um pelo motivo certo.

---

## `issuer-uri` reacoplou a subida — desta vez sabendo

A [Fase 22](../05-seguranca/oauth2-client-e-token.md) trocou `issuer-uri` por `token-uri` no ordering exatamente para a subida de um serviço não depender do authorization server. O gateway reintroduz o acoplamento:

```yaml
spring.security.oauth2.resourceserver.jwt.issuer-uri: http://auth.algashop.local:9000
```

`issuer-uri` dispara a descoberta OIDC **durante o refresh do contexto** — com o AS fora do ar, o gateway não sobe. Para a borda do sistema o trade-off é outro: um gateway de pé sem conseguir validar token não serve a ninguém, e subir quebrado depois seria pior que esperar. O `docker-compose.services.yml` declara a consequência: `depends_on` do `algashop-service-registry` (healthy — sem ele toda rota `lb://` responde 503 até o primeiro fetch) e do `algashop-authorization-server` (started).

---

## Armadilhas

- **O índice do git guardava outro serviço.** O Dockerfile *staged* dizia `ENV JAR_NAME=service-registry.jar` e o `EurekaClientConfig` *staged* tinha o pacote do ordering (`com.gtech...`) — herança do copy-paste que só o working tree corrigiu. Um `git commit` sem novo `git add` subiria um gateway que embala o jar errado. Staged ≠ salvo: o que vai no commit é o índice, não o editor.
- **A config do gateway não vem do Parameter Store.** `issuer-uri` e `defaultZone` estão hardcoded no YAML, duplicando `/config/algashop/shared/auth-server-url` e `/config/algashop/shared/service-registry-url` — os dois parâmetros `shared` que existem exatamente para não serem duplicados. O gateway é hoje o único serviço fora do padrão da [Fase 32](../05-seguranca/segredos-centralizados-e-chave-rsa.md).
- **Porta única por convenção, não por topologia.** Os serviços continuam publicando 8081/8082/8083/9000 no host — o caminho antigo segue aberto, o gateway é a porta *recomendada*. Fechar as outras é o passo que transforma convenção em garantia.
- **`TRACE` no gateway e `DEBUG` no reactor-netty** — cada requisição vira dezenas de linhas. Perfeito para aprender o ciclo predicate→filter→proxy; inaceitável ligado em produção.
- **Perfil `docker` vazio** — como no service-registry, não há `application-docker-env`; o `SPRING_PROFILES_ACTIVE: docker` funciona por ausência de conteúdo.
- **Não há `api.algashop.local`** no `etc/hostnames/hostnames` — o SPA chamaria `localhost:9999`, enquanto a origem liberada no CORS é `admin.algashop.local:4200`. O nome da borda ainda não existe.

## Pendências registradas

- [ ] **Importar a config do Parameter Store** (`shared/auth-server-url`, `shared/service-registry-url`) em vez de duplicá-la — exige o starter da AWS e o `depends_on` do LocalStack.
- [ ] **Fechar as portas dos serviços no host** quando o front migrar para o gateway — porta única de verdade.
- [ ] **Apontar o `webhook-url` do FastPay para o gateway** — a rota `billing-webhook-route` existe e não recebe tráfego.
- [ ] **`api.algashop.local`** no hosts e no CORS/redirects, dando nome à borda.
- [ ] **Baixar o logging** para INFO com chave de ambiente.
- [ ] **Testes: zero** — não há `src/test/`; um `@WebFluxTest`-like das regras de security e um teste de rotas seriam o mínimo.
- [ ] **Healthcheck HTTP real** — o do compose é teste de porta (`/dev/tcp`), pelo mesmo motivo do registry: a imagem temurin-jre não traz curl.

## Checklist de revisão

- [ ] Rota nova: o predicate casa **e** o `authorizeExchange` deixa chegar lá?
- [ ] O path novo pertence a qual serviço — e a ordem das rotas continua do mais específico para o mais genérico?
- [ ] O service ID do `lb://` bate com o `spring.application.name` de quem atende?
- [ ] O endpoint é público? Então `permitAll` nas **duas** camadas — gateway e serviço — e cada uma documentada.
- [ ] A autorização fina (escopo/papel/dono) continua no serviço, ou alguém a "subiu" para o gateway?
- [ ] CORS: a origem nova entrou no `globalcors` do gateway — ou alguém reabriu CORS num serviço de negócio?
- [ ] `/actuator/gateway/routes` mostra a rota como você esperava?

## Referências

- [Spring Cloud Gateway — Route Predicates e Filters](https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway-server-webflux.html)
- [Spring Security — OAuth2 Resource Server (WebFlux)](https://docs.spring.io/spring-security/reference/reactive/oauth2/resource-server/index.html)
- [Microservices.io — API Gateway pattern](https://microservices.io/patterns/apigateway.html)
- [Service discovery](./service-discovery.md) · [Resource servers e escopos](../05-seguranca/resource-server-e-escopos.md) · [OAuth2 client e token](../05-seguranca/oauth2-client-e-token.md) · [Ambiente local](./ambiente-local.md)
