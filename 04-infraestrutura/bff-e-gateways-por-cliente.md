# BFF: a borda se dividiu por cliente — e o front entrou no repositório

> A "porta única" das Fases 34-35 durou duas fases. Este módulo divide a borda **por público** — `api-gateway-ecommerce` (9999) e `api-gateway-admin` (9998) — e traz os dois frontends para dentro do projeto: a SPA do admin e o app server-side do e-commerce, que é o **BFF de verdade**. Cada cliente ganhou o que precisa: rotas próprias, cache onde tolera idade, JSON do tamanho do que renderiza — e a config de todos foi para o Parameter Store, fechando a maior pendência da borda.
> Código real: `microservices/api-gateway-ecommerce` (rename) e `microservices/api-gateway-admin` (novo) — `application-{base,development-env,docker-env,production-env}.yml`, `EcommerceApiController`, `WebClientConfig` · `apps/ecommerce` (Spring + Thymeleaf) e `apps/admin` (Angular 17) · `etc/aws/parameters.csv` e `secrets.csv` · `docker-compose.services.yml` · `build-dev-docker.sh`.
> A borda quando era uma em [API Gateway](./api-gateway.md) · a resiliência dela em [Resiliência na borda](./resiliencia-no-gateway.md) · os fluxos OAuth2 dos dois clientes em [Authorization code](../05-seguranca/authorization-code-e-consentimento.md) e [PKCE](../05-seguranca/pkce-e-clientes-publicos.md).

**Terceiro aviso de vocabulário.** O projeto já desambiguou *gateway de pagamento* (FastPay) de *API gateway*. Falta o terceiro termo: **BFF** (*Backend for Frontend*) não é sinônimo de gateway. BFF é um backend que existe **para um frontend específico** — agrega, adapta e guarda estado por ele. Aqui, o BFF de verdade é o `apps/ecommerce`: um app Spring server-side que guarda o token **em sessão** (Redis), chama a API em nome do usuário e devolve HTML. Os dois gateways são *gateways por público* — parentes do BFF (um por cliente), mas seguem sendo proxies. A única ocorrência literal de "BFF" no código é a flag `algashop.features.home-bff-enabled` do app.

---

## A porta única virou duas — por público, não por acidente

O gateway único servia dois clientes com necessidades opostas. A divisão deixou cada borda com a cara do seu público:

| | `api-gateway-ecommerce` :9999 | `api-gateway-admin` :9998 |
|---|---|---|
| Cliente | `apps/ecommerce` — Spring server-side (Thymeleaf), :9080 | `apps/admin` — SPA Angular 17, :4200 |
| CORS | **nenhum** — server-side não é navegador; o `globalcors` que havia era a origem do *admin*, herdada por copy-paste, e foi removido | `globalcors` com `allowedOrigins` vindo do Parameter Store |
| Cache local | `LocalResponseCache=5m,30MB` no catálogo | **removido** (ver adiante) |
| Rate limit | catálogo, 50/s por `sub` (Redis **db 3**) | catálogo, 50/s por `sub` (Redis **db 5**) |
| Rotas de billing | cartões (`/me/credit-cards`) e webhook do FastPay | fatura administrativa (`/orders/{id}/invoice`) |
| Rotas exclusivas | `/shipping-cost-previews`, webhook | `/orders/**`, `/upload-requests/**` (upload é feature de admin) |
| API composition | `GET /api/v1/ecommerce/home` | não tem |
| JSON otimizado | composição + unwrap do envelope | `RemoveJsonAttributesResponseBody` na listagem |

> **As rotas encolheram para o tamanho do uso real.** O gateway do ecommerce perdeu `/orders/{id}/invoice` administrativa e `/upload-requests` — nenhum call-site do app as usa. Rota que ninguém chama é superfície de ataque de graça; a divisão por público tornou o corte visível.

---

## Dois clientes, duas necessidades de cache

A diferença mais didática entre as bordas é o que **não** está no admin:

- **Ecommerce mantém `LocalResponseCache=5m,30MB`**: o visitante da loja lê um catálogo público; dado com 5 minutos de idade é indistinguível do fresco.
- **Admin perdeu o cache local**: quem opera o catálogo **escreve** nele — e precisa ver a própria escrita. Um PUT seguido de GET devolvendo a versão de 5 minutos atrás pareceria bug, e seria: *read-your-writes* é requisito do público, não do endpoint.

E o recuo foi além da borda: o `ProductController.findById` do catálogo trocou `CacheControl.maxAge(1min).cachePublic()` por **`noCache()`** — o cache client-side da Fase 15 foi desligado pelo mesmo motivo. A [tese do doc de cache](../01-arquitetura-design/cache.md) fecha o raciocínio: a idade de um dado é a **soma** das camadas, e a camada certa para cada resposta depende de *quem* a consome. O mesmo endpoint servindo leitor anônimo e editor não pode ter uma política só — separar as bordas foi o que permitiu separar as políticas.

---

## API composition: o gateway que também responde

O ecommerce gateway ganhou um endpoint próprio — um `@RestController` WebFlux convivendo com o proxy no mesmo processo:

```java
@GetMapping
@PreAuthorize("hasAuthority('SCOPE_products:read') and hasAuthority('SCOPE_categories:read')")
public Mono<Map<String, Object>> getHome() {
    Mono<Map<String, Object>> productList = webClient.get()
            .uri("lb://product-catalog/api/v1/products?hasDiscount=true")
            .retrieve().bodyToMono(...)
            .timeout(Duration.ofSeconds(5))
            .onErrorResume(e -> Mono.just(EMPTY_PAGE));   // ← degrada o ramo, não a home

    Mono<Map<String, Object>> categoriesList = ...        // idem

    return Mono.zip(productList, categoriesList)
            .map(tuple -> Map.of(
                 "highlights", tuple.getT1().getOrDefault("content", List.of()),
                 "categories", tuple.getT2().getOrDefault("content", List.of())));
}
```

O que cada linha carrega:

- **`Mono.zip`** dispara as duas chamadas em paralelo, sem bloquear thread — a home custa o máximo das duas latências, não a soma.
- **`lb://product-catalog`** no `WebClient.Builder` `@LoadBalanced` — a composição resolve instância no Eureka como as rotas proxy.
- **`ServerBearerExchangeFilterFunction`** (no `WebClientConfig`) repassa o Bearer da requisição original — o downstream valida escopo normalmente.
- **`@PreAuthorize` com dois escopos** — a primeira autorização **por escopo** na borda. Até aqui o gateway só autenticava; um endpoint que É do gateway precisa da própria regra.
- **O unwrap do `content`** descarta o envelope de paginação — o client recebe só os dois arrays. Não há DTO no gateway ("o `map` monta a resposta direto em `Map`"); o DTO vive no consumidor (`HomeApiModel`, no app).
- **Timeout e degradação por ramo** entraram nesta fase: categorias fora do ar viram lista vazia com log de warn, **não** um 500 da home inteira. E a armadilha que motivou: **o `httpclient.response-timeout` do gateway não cobre este `WebClient`** — aquela config é do proxy de rotas; a chamada de composição precisa do próprio `.timeout()`. Lista vazia aqui não é o "fallback que mente" da [Fase 16](../01-arquitetura-design/resiliencia.md): ausência é honesta, preço inventado não era.

Do outro lado, o `HomeClient` do app consome o endpoint com `RestClient` síncrono e o `HomeApiModel` de dois campos — as chaves do `Map.of` são o contrato.

---

## JSON do tamanho do cliente: duas técnicas

**Ecommerce — compor e desembrulhar.** A home precisava de dois recursos; a composição entrega um JSON só, sem metadados de paginação, em uma viagem.

**Admin — remover na saída.** A grade de listagem não renderiza `shortDescription` nem `mainImage`, então eles não trafegam:

```yaml
- id: product-catalog-list-route
  uri: lb://product-catalog
  predicates:
    - Path=/api/v1/products      # path EXATO: o /{id} cai na rota geral e recebe o JSON completo
    - Method=GET
  filters:
    - RemoveJsonAttributesResponseBody=shortDescription,mainImage,true   # true = recursivo
```

O filtro é nativo do Spring Cloud Gateway; o `true` final aplica a remoção dentro de cada item de `content[]`. O detalhe fino é a **rota dedicada com path exato**: o detalhe (`GET /api/v1/products/{id}`) continua caindo na rota geral e recebendo tudo.

E a rota dedicada cobrou o preço da ordem — **terceira temporada da mesma lição**: por vir antes da rota geral (primeira que casa vence), a listagem escapava do `RequestRateLimiter`. O endpoint mais chamado da UI era o único sem limite. Corrigido nesta fase: a rota de listagem carrega o próprio rate limiter, com o comentário de ordem gravado no YAML.

---

## A config foi para a AWS — e produção derruba o boot cedo

A pendência aberta na Fase 34 e ampliada na 35 fechou: **11 parâmetros + 2 segredos** nos namespaces `gateway-ecommerce` e `gateway-admin` (Redis, timeouts do httpclient, CORS do admin), mais os dois `shared` de sempre. Os gateways ganharam a mesma estrutura de perfis dos serviços (`base` + `development-env` + `docker-env` + `production-env`), e o `production-env` tem uma decisão que merece registro:

```yaml
  data:
    redis:
      host: ${REDIS_HOST}        # sem default, de propósito
```

Produção não importa da AWS do LocalStack — lê variáveis de ambiente **sem valor default**. Faltou a variável? O boot morre na hora, com nome de propriedade na mensagem. **Fail-fast é a alternativa correta ao default silencioso**: um gateway que subisse apontando para `localhost` em produção falharia longe da causa.

Ficou no YAML o que é desenho, não ambiente: rotas, filtros, resilience, portas. E o compose fechou o elo: os gateways agora dependem de `algashop-localstack: service_healthy` — e "healthy" mudou de significado, ver adiante.

---

## Os dois clientes OAuth, lado a lado

A divisão da borda espelha uma divisão de **modelo de autenticação** que já existia nos clients:

| | `apps/admin` (SPA) | `apps/ecommerce` (BFF) |
|---|---|---|
| Client OAuth | `algashop-admin-web` — **público**, PKCE | `algashop-ecommerce-web` — **confidencial**, `client_secret` |
| Onde vive o token | no navegador (silent refresh por iframe) | **na sessão do servidor** (Redis) — o navegador só vê cookie |
| Fala com o AS | **direto** (login, PKCE, silent refresh — nada disso passa pelo gateway) | direto no bootstrap (descoberta OIDC) e no `authorization_code` |
| Fala com a API | gateway 9998, `Authorization: Bearer` | gateway 9999, Bearer anexado pelo servidor |

O BFF é o motivo de o e-commerce **não precisar** de PKCE: o client é confidencial porque o secret vive num servidor, e o navegador nunca toca o token — a classe de ataque que o PKCE mitiga nem se aplica. Duas mudanças no authorization server acompanharam: o client ecommerce ganhou `invoices:read` (a loja agora mostra a fatura do pedido), e `require-authorization-consent` foi **desligado** para ele — consentimento existe para proteger o usuário de **terceiros**; um client first-party perguntando "você autoriza a nós mesmos?" é atrito sem informação. A tela de consentimento continua lá para quem não for da casa.

---

## O compose aprendeu a ordem de subida — de verdade

Três endurecimentos que este módulo trouxe, cada um consertando uma corrida real:

1. **`authorization-server` e `billing` ganharam healthcheck** (porta aberta = Flyway rodou). Com isso, os `depends_on` dos gateways subiram de `service_started` para `service_healthy` — e o `billing-scheduler` passou a esperar o `billing`, porque num `up` frio ele consultava a tabela `invoice` antes de o Flyway do billing criá-la.
2. **"LocalStack healthy" passou a significar "seed completo"**: o healthcheck trocou a porta pelo `/_localstack/init/ready` com `"completed": true`. Antes, healthy era só porta aberta — e o catálogo chegou a subir **antes de o secret do Redis existir**. A lição: healthcheck mede o que você manda medir, e "de pé" não é "pronto".
3. **Os apps entraram no compose** (`algashop-admin-app` 4200→80 nginx; `algashop-ecommerce-app` 9080 esperando AS e Redis), e nasceu o **`build-dev-docker.sh`** — um script que constrói as 11 imagens locais com três estratégias (docker-compose build para o Angular, `gradlew dockerBuild` para o resto), coleta falhas sem abortar e resume no final.

---

## Armadilhas

- **Três convenções de nome para o mesmo serviço.** O submódulo chama `api-gateway-ecommerce`, o service/imagem do compose chama `algashop-api-gateway`/`api-gateway:dev`, o namespace de parâmetro chama `gateway-ecommerce`. O app do e-commerce chegou a apontar para um quarto nome (`algashop-gateway-ecommerce`) e **reverteu para o nome do compose** — que é o que resolve DNS na rede. Nome de serviço é contrato em três sistemas diferentes; divergir custa um `UnknownHostException` de cada vez.
- **A colisão do db 4 (corrigida).** Os buckets do rate limiter do admin caíram no mesmo banco lógico das **sessões** do ecommerce-app. São naturezas de dado diferentes sob o mesmo `allkeys-lru` — o LRU podia despejar uma sessão para guardar um contador. O admin migrou para o **db 5**; o mapa atual do Redis: 0 cache do catálogo, 1 cache do ordering, 3 buckets do ecommerce-gateway, 4 sessões do ecommerce-app, 5 buckets do admin-gateway.
- **Os `apps/*` estão fora do Parameter Store.** Senha do Redis e client secrets em YAML versionado — a pendência que os gateways acabaram de fechar **renasceu uma camada acima**, nos apps. O padrão existe, os starters existem; falta aplicar.
- **`ecommerce-api.algashop.local` está órfão no hostnames** — nenhum app o usa (o nome vivo é `api.algashop.local`). E `admin-api.algashop.local` chegou a entrar duplicado (removido nesta fase).
- **`environment.prod.ts` do admin declara `production: false`** — o build de produção da SPA rodaria com flags de dev.
- **Código morto por copy-paste entre os gateways**: o admin carrega `WebClientConfig` (ninguém compõe lá), `spring-boot-starter-cache`+Caffeine (nenhum `LocalResponseCache` em rota) e o pacote/classe main sem renomear (`ApiGatewayApplication` nos dois — stacktraces idênticos em logs agregados).
- **As entradas `apps/*` no `.gitmodules` não declaram `branch`** — `git submodule update --remote` se comporta diferente delas.
- **`TRACE`/`DEBUG` seguem em `application-base.yml`** — agora herdados por produção em DOIS gateways.

## Pendências registradas

- [ ] **Config dos `apps/*` no Parameter Store/Secrets Manager** — os secrets dos clients e do Redis seguem em YAML.
- [ ] **`depends_on` do ecommerce-app → gateway 9999** — o app chama um serviço que o compose não garante estar de pé.
- [ ] **Unificar a nomenclatura** (submódulo × service do compose × namespace de parâmetro).
- [ ] **Testes: zero nos dois gateways** — nem composição, nem rotas, nem security.
- [ ] **Baixar o logging dos dois `base`** antes de produção.
- [ ] **Remover o código morto do admin** (WebClientConfig, starter-cache) ou dar-lhe uso.

## Checklist de revisão

- [ ] O recurso novo é de qual público? A rota entra no gateway certo — e só nele?
- [ ] O cliente da rota nova escreve no que lê? Então nada de cache na borda para ele.
- [ ] Rota nova dedicada (path exato, filtro de resposta)? Ela herda os filtros da rota geral? Não — copie o que precisa (a listagem do admin provou).
- [ ] A chamada de composição tem timeout **próprio** e degradação por ramo?
- [ ] O campo que o filtro remove é mesmo dispensável para TODOS os consumidores daquela rota?
- [ ] O client OAuth novo é público ou confidencial — e o token vive onde o modelo manda?
- [ ] Valor novo de config: params/secrets da AWS (dev/docker) e env var **sem default** (produção)?
- [ ] O db do Redis novo está no mapa — e o dado é cache, contador ou sessão?

## Referências

- [Sam Newman — Backends For Frontends](https://samnewman.io/patterns/architectural/bff/)
- [Microservices.io — API Composition](https://microservices.io/patterns/data/api-composition.html)
- [Spring Cloud Gateway — RemoveJsonAttributesResponseBody](https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway-server-webflux/gatewayfilter-factories.html)
- [API Gateway](./api-gateway.md) · [Resiliência na borda](./resiliencia-no-gateway.md) · [Cache](../01-arquitetura-design/cache.md) · [PKCE e clientes públicos](../05-seguranca/pkce-e-clientes-publicos.md) · [Authorization code e consentimento](../05-seguranca/authorization-code-e-consentimento.md) · [Segredos centralizados](../05-seguranca/segredos-centralizados-e-chave-rsa.md)
