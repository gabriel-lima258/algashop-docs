# Resource servers, escopos e a matriz de autorização

> A Fase 20 terminou com a frase "isto ainda não protege nada". Este documento é sobre a outra metade: os três serviços passaram a **exigir** token, cada rota ganhou um escopo, e uma matriz de testes trava a correspondência entre as duas coisas.
> Código real: `infrastructure/security/` (catálogo e billing), `infrastructure/config/security/` (ordering) — `SecurityConfig`, `SecurityAnnotations` e `AuthorizationMatrixTest` em cada um.
> Conceitos em [Identidade e OAuth 2](./fundamentos-identidade-oauth2.md) · o emissor em [Authorization Server](./authorization-server.md).

---

## A outra metade do ciclo

```
                 credenciais            token
   client  ──────────────────▶  AUTHORIZATION SERVER  (:9000)
      │                                  │
      │  ◀─────── JWT ───────────────────┘
      │
      │  Authorization: Bearer eyJ...
      └──────────────────────────▶  RESOURCE SERVER
                                    ordering :8081
                                    billing  :8082
                                    catalog  :8083
```

Três linhas de configuração em cada serviço:

```gradle
implementation 'org.springframework.boot:spring-boot-starter-security-oauth2-resource-server'
```

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: http://auth.algashop.local:9000
```

> **O `issuer-uri` sozinho configura tudo.** Dele o Spring busca `/.well-known/oauth-authorization-server`, e de lá tira o `jwks_uri` — o endereço das chaves públicas. A chave não é copiada para lugar nenhum, e é isso que torna a rotação possível sem deploy coordenado.

O mesmo endereço aparece dos dois lados: `issuer` no emissor, `issuer-uri` em quem valida. Não é redundância — o valor vai dentro do claim `iss` de cada token, e quem valida **compara**. Um token emitido por outro servidor é rejeitado mesmo que a assinatura confira.

---

## A `SecurityConfig`, linha a linha

As três são idênticas (menos uma exceção no billing). Cada decisão vale ser entendida:

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class OrderingSecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .cors(cors -> cors.disable())
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                    .requestMatchers("/actuator/health/**").permitAll()
                    .anyRequest().authenticated())
            .oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()));
        return http.build();
    }
}
```

### `@EnableMethodSecurity` — sem ele as anotações não fazem nada

É o que liga o `@PreAuthorize`. Sem essa linha, as meta-anotações continuam no código, o projeto compila, a suíte passa, e **toda rota fica aberta para qualquer token autenticado**. Falha silenciosa com aparência de sistema protegido.

### Por que desligar CSRF **está certo** aqui

Esta é a que mais gera desconforto, e a resposta é precisa:

> O ataque que o CSRF previne depende de uma credencial que o **navegador envia sozinho** — um cookie de sessão. O site malicioso não precisa ler nada: ele só provoca a requisição, e o navegador anexa o cookie por conta própria.
>
> Um header `Authorization: Bearer` **não é anexado automaticamente por ninguém**. Alguém tem que escrevê-lo, e para escrevê-lo precisa ter o token. O ataque não existe.

Desligar CSRF numa API com token é correto. Desligar numa aplicação com sessão e cookie é uma vulnerabilidade. **A mesma linha é certa num caso e grave no outro** — e é por isso que ela não deve ser copiada sem entender qual dos dois se está construindo.

### `STATELESS` — e a razão de as duas decisões andarem juntas

Nenhuma sessão, nenhum `JSESSIONID`. Cada requisição se identifica sozinha pelo token, o que permite escalar horizontalmente sem sessão compartilhada. E é justamente por não haver sessão que o parágrafo anterior vale: sem cookie de sessão, não há o que o navegador envie sozinho.

### `permitAll` e a armadilha do caminho literal

A versão que entrou primeiro era:

```java
.requestMatchers("/actuator/health").permitAll()
```

`requestMatchers` com caminho literal casa **exatamente** aquele path. `/actuator/health/readiness` e `/actuator/health/liveness` são **subpaths** — eles caíam em `anyRequest().authenticated()` e respondiam **401**.

> A [Fase 17](../04-infraestrutura/health-checks.md) construiu o grupo `readiness` precisamente para o orquestrador consultar antes de mandar tráfego. Com 401, a instância nunca entra em rotação — e o sintoma é um deploy que "não sobe", sem nenhum erro na aplicação.

A correção é `/actuator/health/**`, e há um teste fixando os três caminhos em cada serviço.

### `anyRequest().authenticated()` — fecha por padrão

Rota nova nasce protegida. O inverso — liberar por padrão e lembrar de proteger — erra em silêncio, e o erro só aparece quando alguém já entrou.

### A exceção do billing

```java
.requestMatchers("/api/v1/webhooks/**").permitAll()
```

O FastPay chama de fora e não carrega token — não há como exigir. É a única rota deliberadamente aberta do sistema, e tem um teste que **fixa** essa abertura, para que ninguém a "conserte" achando que é descuido e quebre a confirmação de pagamento.

---

## 401 × 403

| | Significa | Quem decide |
|---|---|---|
| **401** Unauthorized | *não sei quem você é* — token ausente, inválido ou expirado | a cadeia de filtros |
| **403** Forbidden | *sei quem você é, e você não pode* | o `@PreAuthorize` |

O nome do 401 é historicamente infeliz: ele deveria se chamar *Unauthenticated*. A regra prática: **401 pede credencial, 403 diz que a credencial não basta.**

```java
@ExceptionHandler(AccessDeniedException.class)
public ProblemDetail handleAccessDeniedException(AccessDeniedException e) {
    ProblemDetail problemDetail = ProblemDetail.forStatus(HttpStatus.FORBIDDEN);
    problemDetail.setTitle("Forbidden");
    problemDetail.setDetail("You do not have permission to access this resource");
    problemDetail.setType(URI.create("/errors/forbidden"));
    return problemDetail;
}
```

> Repare no que a mensagem **não** diz: qual escopo faltou. É deliberado. Responder "faltou `invoices:write`" transforma o 403 num mapa da superfície de autorização do sistema — quem sonda descobre os nomes dos escopos sem ter nenhum. O detalhe útil vai para o log, não para o corpo.

Ver [tratamento de erros](../03-testes-integracao/tratamento-erros-api.md).

---

## A ordem das camadas — e a surpresa

Construir a matriz revelou uma ordem que não é a intuitiva:

```
1. cadeia de filtros (autenticação)     -> 401
2. resolução de argumentos + @Valid     -> 400
3. @PreAuthorize (method security)      -> 403
```

`@PreAuthorize` é um proxy AOP em volta do **método** do controller. Para chamá-lo, o Spring MVC precisa antes resolver os argumentos — e é aí que `@Valid` roda. **Um corpo inválido responde 400 antes de a autorização ser consultada.**

Medido, com um token que não tem escopo nenhum relevante:

```
POST /api/v1/customers  {}   com SCOPE_totally:unrelated   ->  400   (não 403)
POST /api/v1/customers  <corpo válido>  com o mesmo token  ->  403
```

Duas consequências:

**Prática:** a matriz de testes precisa enviar corpos válidos, senão o caso de 403 nunca é exercitado — ele morre em 400 antes. Foi assim que a primeira versão falhou em 6 dos 18 casos.

**De segurança:** um chamador sem permissão nenhuma consegue descobrir o formato esperado do payload. Não é vazamento grave — a mensagem fala de campos, não de dados —, mas contraria a expectativa de que 403 vem primeiro. Mover a decisão para `authorizeHttpRequests` (no filtro, por path) inverteria a ordem, ao custo de perder a expressividade da anotação junto ao método. Fica registrado como pendência.

---

## `SCOPE_` e as meta-anotações

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@PreAuthorize("hasAuthority('SCOPE_orders:read')")
public @interface CanReadOrders {}
```

```java
@CanReadOrders
@GetMapping
public PageModel<OrderSummaryOutput> findAll(OrderFilter filter) { ... }
```

**De onde vem o prefixo:** o `JwtGrantedAuthoritiesConverter`, padrão do resource server, lê o claim `scope` do token e prefixa cada valor com `SCOPE_` ao transformá-lo em `GrantedAuthority`. Por isso `hasAuthority('SCOPE_orders:read')` e não `hasScope('orders:read')`.

**Por que a meta-anotação, e não `@PreAuthorize` solto:**

A expressão do `@PreAuthorize` é uma **string avaliada em runtime**. Um typo — `'SCOPE_orders'` sem o sufixo, ou `'SCOPE_orders:raed'` — compila, sobe, e **nega todo mundo em silêncio**. Nenhum compilador ajuda.

Concentrar as strings num arquivo reduz a superfície de erro de N controllers para um lugar, e é o que torna a matriz de testes capaz de cobri-las. Como bônus, o controller passa a declarar a intenção (`@CanReadOrders`) em vez da sintaxe.

### Nem toda meta-anotação verifica escopo

A Fase 25 acrescentou um tipo diferente, ao lado das de escopo:

```java
@PreAuthorize("hasAuthority('SCOPE_users:read')")        // permissão: o que o token pode
public @interface CanReadUsers {}

@PreAuthorize("@securityCheck.canAccessOwnProfile()")    // sujeito: quem o token representa
public @interface CanAccessOwnProfile {}
```

A segunda não pergunta *o que este token autoriza*, e sim *este token representa uma pessoa?* — porque `/api/v1/users/me` não faz sentido para um token de máquina. Máquina recebe **403**: não é falta de permissão, é ausência de sujeito.

E ela agrava a armadilha desta seção. `@securityCheck` é resolvido **pelo nome do bean** (`@Service("securityCheck")`), em runtime, dentro de uma string. Renomear o bean e esquecer a expressão quebra a autorização sem erro de compilação, sem aviso da IDE e sem log — o método simplesmente passa a negar todo mundo. Detalhes em [Gestão de usuários e auditoria](./gestao-de-usuarios-e-auditoria.md).

### O desenho dos escopos

| Serviço | Escopos |
|---|---|
| `product-catalog` | `products:read`, `products:write`, **`products:stock:write`**, `categories:read`, `categories:write` |
| `ordering` | `orders:read/write`, `customers:read/write`, `shopping-carts:read/write`, `shipping-costs:preview` |
| `billing` | `invoices:read/write`, `credit-cards:read/write` |
| `authorization-server` | `users:read`, `users:write` *(Fase 25)* |

Duas escolhas valem ser notadas:

**`products:stock:write` é separado de `products:write`.** Quem integra estoque não ganha de brinde o direito de reescrever preço. Há um teste que prova que `products:write` **não** abre `/restock` nem `/withdraw`.

**O carrinho do cliente exige escopo de carrinho, não de cliente.** `GET /api/v1/customers/{id}/shopping-cart` mora sob `/customers` mas pede `shopping-carts:read`. O escopo segue o **recurso**, não a URL — e é exatamente o tipo de decisão que ninguém lembra seis meses depois, e que um teste fixa.

---

## A matriz de autorização como teste

Para **cada** rota anotada, três perguntas:

```java
@WebMvcTest(controllers = { OrderController.class, CustomerController.class, ... })
@Import(OrderingSecurityConfig.class)
class AuthorizationMatrixTest {

    @MockitoBean private JwtDecoder jwtDecoder;
    // ... application services mockados

    static Stream<Arguments> routes() {
        return Stream.of(
            Arguments.of(HttpMethod.GET,  "/api/v1/orders", "SCOPE_orders:read",  null, null),
            Arguments.of(HttpMethod.POST, "/api/v1/orders", "SCOPE_orders:write", ORDER_WITH_PRODUCT, body()),
            // ...
        );
    }
}
```

Quatro decisões de desenho que valem mais que o código:

**1. Importar a `SecurityConfig` real não é opcional.** Sem `@Import`, o `@WebMvcTest` autoconfigura a cadeia de filtros **padrão do Spring Boot** — e o teste passaria a afirmar sobre uma configuração que o projeto não usa. O `@EnableMethodSecurity` também vem de lá; sem ele, o `@PreAuthorize` não roda e todo caso de 403 falha.

**2. O `@MockitoBean JwtDecoder` existe só para o contexto subir.** `oauth2ResourceServer().jwt()` exige o bean. Ele nunca é chamado, porque o post-processor `jwt()` do `spring-security-test` injeta a autenticação já pronta.

**3. A asserção positiva é "não é 401 nem 403", não "é 200".** Um teste de segurança deve falhar quando a **segurança** muda — não quando o controller passa a devolver 404 por causa de um id inexistente. Acoplar ao resultado de negócio transforma a matriz numa fonte de falha por motivo errado.

**4. O escopo "errado" é um escopo que não existe** (`SCOPE_totally:unrelated`). Isso prova o ponto que interessa: **estar autenticado não basta**. O token é válido, o portador é conhecido, e ainda assim leva 403.

### Números

| Serviço | Rotas na matriz | Testes de segurança |
|---|---|---|
| `ordering` | 18 | **58** |
| `product-catalog` | 19 | **62** |
| `billing` | 6 | **22** |

Antes desta fase eram **2** no sistema inteiro, ambos no `CustomerControllerIT`. O `billing` e o `product-catalog` não tinham nenhum teste de camada HTTP — esta é a primeira.

### A matriz tem dentes — verificado

Um teste que nunca falha não prova nada. Comentando `@CanReadOrders` do `GET /api/v1/orders`:

```
AuthorizationMatrixTest > GET "/api/v1/orders" com escopo errado -> 403 FAILED
```

Exatamente um caso vermelho, exatamente o certo.

---

## Os dois níveis de teste, e o que cada um cobre

| | `AuthorizationMatrixTest` (fatia) | `*ControllerIT` (Testcontainers) |
|---|---|---|
| Sobe | só a camada web | contexto inteiro, Postgres real, WireMock |
| Token | `jwt()` post-processor | `MockJwtDecoderFactory` + header real |
| Cobre | a **matriz** rota × escopo | a **fiação** ponta a ponta |
| Custo | segundos | dezenas de segundos |

> ⚠️ **Nenhum dos dois testa autenticação de verdade.** O mock substitui o `JwtDecoder` inteiro: assinatura, `iss`, `exp` e `aud` nunca são verificados. E o caso "expirado" do `MockJwtDecoderFactory` é um `thenThrow(JwtException)` — exercita o tratamento de falha de decode, não um token vencido sendo rejeitado pelo validador.
>
> Isso não é defeito, é o recorte certo: testar validação de JWT seria testar o Spring Security. Mas precisa estar escrito, senão a suíte verde sugere uma cobertura que não existe. **O que está coberto é autorização.**

---

## O achado: o `ordering` chamava o catálogo sem token

> ✅ **Fechado na Fase 22.** O `ordering` virou OAuth2 client, obtém token por `client_credentials` e anexa o `Bearer` por interceptor; e o `catch` foi estreitado para `NotFound`, então 401 deixou de virar 422. O que segue é o registro do problema, que continua valendo como lição. Ver [OAuth2 client e token](./oauth2-client-e-token.md).

O catálogo agora exige autenticação em tudo. O `ordering` chama o catálogo por HTTP a cada compra. E não há uma linha de propagação de token no client de saída — nenhum `Authorization`, nenhum `ClientRegistration`.

O que acontece é pior do que uma falha:

```
ordering → catálogo            -> 401
ResilientProductCatalogAPIClient mapeia 4xx para Optional.empty()
        → ProductNotFoundException
        → HTTP 422 "produto não encontrado"
```

> **Uma falha de configuração de segurança se apresenta como erro de negócio.** É o pior tipo de bug para diagnosticar: o sintoma aponta para o catálogo de produtos, e a causa está na ausência de um header.

O cliente `algashop-ordering-service`, com escopo `products:read` e TTL de 5 minutos, **já está registrado** no authorization server esperando ser usado. Falta o `ordering` pedir o token (`spring.security.oauth2.client`) e anexá-lo às chamadas de saída.

Vale registrar o mecanismo geral, porque ele se repete: **engolir 4xx é conveniente até o dia em que a causa do 4xx muda.** O `Optional.empty()` foi escolhido quando 404 era a única razão plausível; com autenticação no caminho, ele passou a esconder um 401.

---

## O `sub` mudou de significado na Fase 24

Nos fluxos com usuário (`authorization_code` e `refresh_token`), o access token passou a trazer o **UUID do `auth_user`** no `sub`, em vez do e-mail:

```
sub = 019d7764-3b02-7fd5-b0e7-c47c58592857     (era john.doe@email.com)
```

Para os resource servers isso é uma **mudança de contrato** — e uma oportunidade. É a primeira vez que existe um identificador estável de pessoa atravessando a fronteira dos serviços, o que dá destino à pendência mais antiga do caderno: a auditoria do `product-catalog`, que grava um `UUID` aleatório como autor.

> Em `client_credentials` nada muda: não há pessoa, e o `sub` continua sendo o `client_id`. Quem for ler o `sub` precisa saber **qual** dos dois tipos de token está recebendo.

✅ **A Fase 25 usou.** Os três serviços que auditam trocaram o `UUID.randomUUID()` pelo `sub`, e a pergunta "qual dos dois tipos de token é este?" ganhou uma resposta explícita — `isMachineAuthenticated()`, que compara `aud` e `sub`. Verificado nos dois casos:

```
máquina:  sub = algashop-test                aud = algashop-test           → aud contém sub
usuário:  sub = 019d7764-3b02-7be2-9112-…    aud = algashop-ecommerce-web  → não contém
```

Ver [OpenID Connect: identidade, sessão e logout](./openid-connect-e-sessao.md) e [Gestão de usuários e auditoria](./gestao-de-usuarios-e-auditoria.md).

---

## Armadilhas

- **`@EnableMethodSecurity` ausente** deixa as anotações decorativas, sem erro nenhum.
- **`requestMatchers` com caminho literal** não cobre subpaths.
- **`@Valid` roda antes do `@PreAuthorize`** — 400 vem antes de 403.
- **Typo em escopo nega para sempre**, em silêncio.
- **Contract tests não veem segurança.** As bases montam o MockMvc com `webAppContextSetup(context)` **sem** `.apply(springSecurity())`: a cadeia de filtros não é aplicada, e o `contractTest` passa verde mesmo com todos os endpoints protegidos. O contrato publicado continua descrevendo uma API sem autenticação.
- **Desligar CSRF** é certo aqui e grave numa aplicação com sessão.

---

## Pendências registradas

- [x] ~~**O `ordering` não propaga token ao catálogo.**~~ Fechado na Fase 22: interceptor OAuth2 no `RestClient`, com teste que fica vermelho se ele sair. E o 4xx deixou de ser engolido em bloco — 401 agora vira 502. Ver [OAuth2 client e token](./oauth2-client-e-token.md).
- [ ] **Nenhum resource server valida `aud`.** Um token emitido para o `algashop-ordering-service` é aceito pelo billing e pelo catálogo — o escopo limita *o que* ele faz, nada limita *onde* ele vale. Um serviço comprometido replica contra os outros os tokens que recebeu. Fechar exige decidir a audiência de cada serviço e acrescentar um `OAuth2TokenValidator`.
- [ ] **O webhook do FastPay muda estado de fatura sem verificar origem.** Público é necessário; sem verificação não deveria ser. Assinatura HMAC ou allowlist de IP resolveriam — hoje **qualquer um marca fatura como paga**.
- [ ] **O issuer é `http://` e é nome de container.** Ele vai dentro do `iss` de todo token; trocá-lo invalida tudo em circulação. Em produção precisa ser https e estável.
- [ ] **A validação de token nunca é exercitada em teste.** Assinatura, `iss` e `exp` só são verificados de verdade contra o authorization server rodando, e nada faz isso hoje.
- [ ] **O authorization server não está no compose**, e todos os perfis apontam para o nome de container dele. Sobe por `./gradlew bootRun`.
- [ ] **Os contract tests deveriam aplicar `springSecurity()`** — ou o contrato deveria declarar que os endpoints exigem token.
- [ ] **`@Valid` antes de `@PreAuthorize`** — mover a autorização para o filtro inverteria a ordem, ao custo da expressividade.
- [ ] **As três `SecurityConfig` são cópias.** Aceitável entre serviços independentes; vira problema quando uma delas divergir por engano em vez de por decisão.

---

## Checklist de revisão

- [ ] `@EnableMethodSecurity` está presente?
- [ ] Os `permitAll` cobrem subpaths quando deveriam (`/**`)?
- [ ] A rota nova nasce coberta por `anyRequest().authenticated()`?
- [ ] O escopo novo entrou na matriz de teste?
- [ ] O escopo é o **mais estreito** que resolve o caso?
- [ ] Cada rota deliberadamente aberta tem um teste que **fixa** a abertura?
- [ ] O 403 evita dizer qual permissão faltou?
- [ ] Quem chama outro serviço propaga token?

---

## Referências

- [Spring Security — OAuth2 Resource Server](https://docs.spring.io/spring-security/reference/servlet/oauth2/resource-server/index.html)
- [Spring Security — Method Security](https://docs.spring.io/spring-security/reference/servlet/authorization/method-security.html)
- [Spring Security — Testing OAuth 2.0](https://docs.spring.io/spring-security/reference/servlet/test/mockmvc/oauth2.html)
- [Identidade e OAuth 2](./fundamentos-identidade-oauth2.md) · [Authorization Server](./authorization-server.md)
- [Health check e degradação](../04-infraestrutura/health-checks.md) — o readiness que o `permitAll` literal derrubava
