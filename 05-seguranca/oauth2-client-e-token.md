# O `ordering` como OAuth2 client: obtenção, cache e propagação de token

> A Fase 21 fez os três serviços **exigirem** token e terminou com um achado: o `ordering` chamava o catálogo **sem** token, levava 401, e entregava ao usuário um **422 "produto não encontrado"**. Este documento é sobre a outra metade — quem **pede** o token — e sobre o acoplamento de inicialização que veio junto e foi desfeito.
> Código real: `infrastructure/config/security/OAuth2ClientConfig.java`, `adapters/out/web/product/client/http/ProductCatalogApiConfig.java` e `ProductCatalogIntegrationProperties.java`, `application-development-env.yaml` (ordering).
> Conceitos em [Identidade e OAuth 2](./fundamentos-identidade-oauth2.md) · o emissor em [Authorization Server](./authorization-server.md) · quem valida em [Resource servers e escopos](./resource-server-e-escopos.md).

---

## O papel duplo

O `ordering` é as duas coisas ao mesmo tempo, e as duas configurações são independentes:

```
                            ┌──────────────────────────────┐
   cliente  ──token──▶      │          ORDERING            │
   (exige token)            │                              │
                            │  resource server: VALIDA     │  ← OrderingSecurityConfig
                            │  client:          PEDE       │  ← OAuth2ClientConfig
                            └──────────────────────────────┘
                                    │              │
                          pede token│              │chama com token
                                    ▼              ▼
                        AUTHORIZATION SERVER   PRODUCT-CATALOG
```

> Ser resource server não torna um serviço capaz de chamar outro serviço protegido. São blocos de configuração diferentes, dependências diferentes (`...-resource-server` e `...-oauth2-client`) e ciclos de vida diferentes. Confundir os dois é o que faz alguém "já ter configurado OAuth2" e ainda assim tomar 401 na chamada de saída.

---

## As três peças

Configurar isso dá errado quase sempre pelo mesmo motivo: falta **uma** das três, e as outras duas parecem completas.

| # | Peça | Responde |
|---|---|---|
| 1 | o `registration` no YAML | **como** obter o token — endereço, id, segredo, grant, escopo |
| 2 | o `OAuth2AuthorizedClientManager` | **executar** o `client_credentials` e guardar o resultado |
| 3 | o `OAuth2ClientHttpRequestInterceptor` | **anexar** o `Authorization` em cada requisição |

Sem a **1**, o Spring não cria `ClientRegistrationRepository`. Sem a **2**, ninguém busca. **Sem a 3, tudo está configurado, a aplicação sobe sem um aviso, e nenhuma chamada leva header.** Era exatamente esse o estado na Fase 21.

### 1 — o registration

```yaml
spring:
  security:
    oauth2:
      client:
        provider:
          algashop-as:
            token-uri: ${algashop.integrations.authorization-server.url}/oauth2/token
        registration:
          algashop-ordering-service-client:
            provider: algashop-as
            client-id: algashop-ordering-service
            client-secret: secret123
            authorization-grant-type: client_credentials
            client-authentication-method: client_secret_basic
            scope: products:read
```

O `client-id` e o escopo casam com o cliente que o [authorization server](./authorization-server.md) já tinha registrado desde a Fase 20 — **`products:read` e nada mais**, porque o `ordering` só lê o catálogo.

### 2 — o manager

```java
@Bean
public OAuth2AuthorizedClientManager auth2AuthorizedClientManager(
        ClientRegistrationRepository clientRegistrationRepository,
        OAuth2AuthorizedClientService oAuth2AuthorizedClientService) {

    var provider = OAuth2AuthorizedClientProviderBuilder.builder()
            .clientCredentials()
            .build();

    var manager = new AuthorizedClientServiceOAuth2AuthorizedClientManager(
            clientRegistrationRepository, oAuth2AuthorizedClientService);
    manager.setAuthorizedClientProvider(provider);
    return manager;
}
```

**`AuthorizedClientService...` e não `Default...`** — e a diferença importa. O `DefaultOAuth2AuthorizedClientManager` resolve o cliente autorizado a partir da **requisição HTTP em curso**: ele espera `HttpServletRequest` e o `SecurityContext` da thread. Aqui não há requisição de usuário envolvida; é máquina falando com máquina, e o mesmo código precisa funcionar num job agendado ou num pool assíncrono. A variante de serviço guarda o token num `OAuth2AuthorizedClientService`, sem depender de contexto web nenhum.

**O provider também renova.** Antes de devolver um cliente autorizado, o `ClientCredentialsOAuth2AuthorizedClientProvider` confere a expiração — com uma folga de relógio — e busca outro token se estiver vencido. É por isso que o TTL de 5 minutos do registration **não** custa uma ida ao authorization server por requisição.

### 3 — o interceptor

```java
var interceptor = new OAuth2ClientHttpRequestInterceptor(manager);
interceptor.setClientRegistrationIdResolver(_ -> properties.getOauth2ClientRegistrationId());
interceptor.setPrincipalResolver(_ -> generatePrincipal(properties.getOauth2ClientRegistrationId()));

RestClient restClient = builder.baseUrl(properties.getUrl())
        .requestFactory(generateClientHttpRequestFactory())
        .requestInterceptor(interceptor)
        .build();
```

O `clientRegistrationIdResolver` responde *qual* registration usar. Não é dedutível da URL de destino — a escolha é de configuração, e por isso o `oauth2ClientRegistrationId` viaja junto com a `url` no mesmo `@ConfigurationProperties`: **endereço e credencial de acesso são a mesma decisão de integração**, e separá-los deixaria um mudar sem o outro.

---

## O principal sintético — a parte mais fina

```java
private Authentication generatePrincipal(String principalName) {
    return new AbstractAuthenticationToken(Collections.emptySet()) {
        public Object getPrincipal()   { return principalName; }
        public Object getCredentials() { return null; }
    };
}
```

Um `Authentication` de fachada, com authorities vazias, credenciais nulas, que **nunca entra no `SecurityContext`**. Parece gambiarra e não é — é a correção de um problema real.

O manager indexa o token pelo par **`(registrationId, principalName)`**. O resolver padrão usa o `Authentication` da thread, e nas duas situações que este serviço tem isso dá errado:

| Situação | Principal padrão | Consequência |
|---|---|---|
| requisição HTTP de um usuário | o JWT de quem chamou o `ordering` | um token **de máquina** cacheado **por usuário** — o cache fragmenta e cada usuário novo custa uma ida ao `/oauth2/token` |
| job, listener, pool assíncrono | `anonymousUser` ou nada | chave instável, ou nenhuma |

Com um principal constante, todas as chamadas compartilham a **mesma** entrada de cache.

> Vale generalizar: **a identidade que autentica e a identidade que indexa um cache não são a mesma coisa.** O `principalResolver` só nomeia o dono da entrada; misturá-lo com a autenticação da requisição faz um token que não pertence a ninguém ser guardado como se pertencesse a alguém.

Verificado por teste — duas chamadas ao catálogo custam **no máximo uma** ida ao `/oauth2/token`:

```java
client.getById(EXISTING_PRODUCT);
client.getById(NOT_FOUND_PRODUCT);

wireMockProductCatalog.verify(lessThanOrExactly(1),
        postRequestedFor(urlEqualTo("/oauth2/token")));
```

---

## `token-uri` × `issuer-uri`: o acoplamento de inicialização

Antes desta fase o `ordering` **não subia sem o authorization server no ar**. A causa está numa escolha de uma linha:

```yaml
provider:
  algashop-as:
    token-uri: ${...}/oauth2/token      # explícito — sem descoberta
    # issuer-uri: ${...}                # descoberta na SUBIDA do contexto
```

> `spring.security.oauth2.client.provider.<x>.issuer-uri` faz o Spring executar `ClientRegistrations.fromIssuerLocation()` **na criação do bean** — uma ida ao `/.well-known` durante o refresh do contexto. Se o authorization server não responde, a aplicação não inicia. E o sintoma é o pior possível: um serviço que "não sobe", sem relação aparente com segurança.

Declarando `token-uri`, não há descoberta. O endereço já é conhecido, e a primeira ida à rede passa a ser **a primeira requisição que precisar de token** — que falha sozinha, com o circuito e o retry já existentes ao redor.

O preço é perder a descoberta automática dos outros endpoints do authorization server. `client_credentials` não usa nenhum deles: só o `/oauth2/token`.

### E por que o lado resource server pode continuar com `issuer-uri`

Porque as duas metades resolvem o endereço em momentos diferentes. O `issuer-uri` do **resource server** serve para validar tokens que chegam, e a resolução das chaves acontece na primeira validação — não na subida. O do **client** é que era eager.

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: ${algashop.integrations.authorization-server.url}
      client:
        provider:
          algashop-as:
            token-uri: ${algashop.integrations.authorization-server.url}/oauth2/token
```

**Um endereço, duas leituras.** `algashop.integrations.authorization-server.url` é a fonte única: os dois papéis interpolam de lá. Ter que mudar o valor em dois lugares seria um jeito silencioso de deixá-los divergir — e um resource server validando contra um issuer diferente do que o client usa para pedir token recusa todo token que ele mesmo mandou buscar.

> ⚠️ O comportamento descrito aqui foi verificado em execução **pelo autor do projeto**, ao fazer a mudança. Este documento não reencena a medição: os números que aparecem abaixo são de suíte de testes, não de serviços de pé.

---

## O achado da Fase 21, fechado

O `ResilientProductCatalogAPIClient` engolia todo 4xx:

```java
// antes
} catch (HttpClientErrorException e) {
    if (!(e instanceof HttpClientErrorException.NotFound)) {
        log.error("Client HTTP error when loading product {}", productId, e);
    }
    return Optional.empty();
}

// depois
} catch (HttpClientErrorException.NotFound e) {
    return Optional.empty();
}
```

Com o `catch` estreitado, só o **404** vira vazio. 401 e 403 caem em `translateException` e viram `BadGatewayException.ClientErrorException` → **502**, sem retry.

| | antes | depois |
|---|---|---|
| 404 | `Optional.empty()` → 422 | igual |
| **401 / 403** | `Optional.empty()` → **422 "produto não encontrado"** | **502** |

> A lição não é sobre OAuth: **engolir uma classe inteira de erro é conveniente até o dia em que a causa muda.** O `Optional.empty()` foi escolhido quando 404 era a única razão plausível de um 4xx; quando autenticação entrou no caminho, ele passou a esconder um 401 e a apontar o dedo para o catálogo de produtos.

O teste que fixava o comportamento antigo trazia um `TODO` prevendo exatamente isso — *"Se o mapeamento mudar, este teste muda junto"* — e foi reescrito para afirmar o novo desfecho.

### E `unless = "#result == null"`

```java
@Cacheable(cacheNames = "algashop:product-catalog-api:v1", key = "#productId",
           unless = "#result == null")
```

O Spring **desembrulha** o `Optional` antes de gravar no cache, então um resultado vazio vira `null`, e o cache tem `disableCachingNullValues()` — o `put` era recusado com um `WARN` a cada 404. O `unless` correto é `#result == null` e não `#result.isEmpty()`, porque o mesmo desembrulho vale para a expressão do `unless`: `#result` já é o `ProductResponse`, não o `Optional`.

---

## O que os testes provam

Dois testes novos no `ResilientProductCatalogAPIClientIT`, com o `/oauth2/token` servido pelo mesmo WireMock que finge ser o catálogo:

```java
client.getById(EXISTING_PRODUCT);

wireMockProductCatalog.verify(getRequestedFor(urlEqualTo("/api/v1/products/" + EXISTING_PRODUCT))
        .withHeader("Authorization", equalTo("Bearer fake-machine-token")));
```

O valor do header é o que o `/oauth2/token` devolveu — **ele não foi montado à mão em lugar nenhum do código**. Veio do fluxo `client_credentials` inteiro.

**Verificado que o teste tem dentes:** removendo `.requestInterceptor(interceptor)` do `ProductCatalogApiConfig`, exatamente um teste fica vermelho.

Suíte do `ordering`: **480 testes, 0 falhas** (eram 478).

---

## O padrão que se repete: a árvore de configuração esquecida

Esta é a **terceira fase seguida** em que uma propriedade obrigatória entra no `development-env` e a suíte quebra:

| Fase | O que entrou | Como quebrou |
|---|---|---|
| 19 | `storage.provider`, `image-storage-url` | contexto não subia por falta de região da AWS |
| 21 | `issuer-uri` / `JwtDecoder` | `SecurityFilterChain` sem `JwtDecoder` |
| 22 | registration de client | `SecurityFilterChain` sem `ClientRegistrationRepository` |

> `src/test/resources` é uma árvore de configuração **separada**, que ninguém lembra de visitar ao acrescentar uma propriedade obrigatória. E o sintoma nunca aponta para a causa: o erro fala de um bean que falta, não de uma linha de YAML que não foi copiada.

O que reduziria isso: `@ConfigurationProperties` com `@NotBlank` só quando a funcionalidade está ativa (`@ConditionalOnProperty`, como o `StorageProvider` do catálogo faz), em vez de obrigatório em todo perfil.

---

## Armadilhas

- **Configurar as três peças menos o interceptor** — sobe sem aviso e nenhuma chamada leva header.
- **`issuer-uri` no lado client** — acopla a subida da aplicação ao authorization server.
- **Principal padrão no cache de token** — fragmenta por usuário, ou some em thread sem `SecurityContext`.
- **`.oauth2Client()` na `SecurityFilterChain`** liga os filtros de *authorization code* (redirect e callback), que não fazem nada em `client_credentials` — e faz a cadeia exigir `ClientRegistrationRepository`, quebrando qualquer `@WebMvcTest` sem registration configurada.
- **Engolir 4xx inteiro** — esconde 401 e 403 atrás de "não encontrado".
- **`unless = "#result.isEmpty()"`** estoura: o `#result` já vem desembrulhado.

---

## Pendências registradas

- [x] ~~**O `ordering` não propaga token ao catálogo.**~~ Fechado nesta fase, com teste que falha se o interceptor sair.
- [ ] **`client-secret: secret123` em texto puro** num arquivo versionado — o par do `{noop}` do lado do authorization server. Mínimo aceitável: variável de ambiente ou cofre.
- ✅ ~~**No perfil `docker` o `ordering` não alcança o authorization server.**~~ — **resolvido na Fase 26**: o AS entrou no `docker-compose.services.yml` e responde por `auth.algashop.local` dentro da rede (o DNS do Docker resolve o `hostname:` do container).
- [ ] **O `.oauth2Client()` da `SecurityFilterChain` não tem uso hoje.** Ele só passa a ser necessário se o `ordering` ganhar login de usuário com `authorization_code`.
- [ ] **O cache de token é por instância** (`InMemoryOAuth2AuthorizedClientService`). Com N réplicas, são N tokens em circulação — aceitável, mas multiplica a carga no authorization server proporcionalmente.
- [ ] **Nada testa a renovação.** O provider renova quando o token expira; nenhum teste exercita um token vencido.
- [ ] **O `billing` não é client.** Se um dia ele precisar chamar outro serviço protegido, esta configuração inteira se repete lá — e o momento de extrair o padrão é esse, não antes.

---

## Checklist de revisão

- [ ] As três peças existem — registration, manager e **interceptor**?
- [ ] O `token-uri` é explícito, para não acoplar a subida?
- [ ] O endereço do authorization server tem uma fonte única?
- [ ] O `principalResolver` é constante, e não o `Authentication` da requisição?
- [ ] O escopo pedido é o mínimo que o serviço precisa?
- [ ] O tratamento de erro distingue "não encontrado" de "não autorizado"?
- [ ] A propriedade obrigatória nova foi copiada para `src/test/resources`?

---

---

## 🔄 O interceptor virou bean próprio (Fase 29)

Ele nascia dentro do método que criava o `ProductCatalogApiClient` — e por isso **não havia como substituí-lo em teste**. Todo IT de apresentação tentava buscar token de verdade no authorization server, que não está de pé durante os testes.

```java
@Bean("productCatalogAPIClientInterceptor")
public OAuth2ClientHttpRequestInterceptor productCatalogAPIClientInterceptor(...) { ... }
```

Mesmo objeto, mesmo lugar no fluxo, agora com nome — e os testes o trocam por um `@MockitoBean`.

> **Dependência criada dentro de um método não é um bean, não tem nome, e nada externo a alcança.** Testabilidade acabou sendo propriedade do desenho, não do teste. Ver [Testando segurança](../03-testes-integracao/testando-seguranca.md#testabilidade-é-propriedade-do-desenho).

## Referências

- [Spring Security — OAuth2 Client](https://docs.spring.io/spring-security/reference/servlet/oauth2/client/index.html)
- [Spring Security — `client_credentials` grant](https://docs.spring.io/spring-security/reference/servlet/oauth2/client/authorization-grants.html#_client_credentials)
- [Spring Security — RestClient integration](https://docs.spring.io/spring-security/reference/servlet/oauth2/client/authorized-clients.html)
- [Resource servers e escopos](./resource-server-e-escopos.md) — a outra metade
- [Resiliência](../01-arquitetura-design/resiliencia.md) — o circuito e o retry ao redor desta chamada
