# OpenID Connect: identidade, sessão e logout

> A Fase 23 trouxe a pessoa para dentro do fluxo, mas o sistema só sabia **o que ela autorizou** — não **quem ela é**. Este documento é sobre a camada de identidade, e sobre as duas coisas que vêm junto com ela: um usuário de verdade no banco, e sessão que sobrevive a um restart.
> Código real: `domain/AuthUser.java`, `infrastructure/security/` — `AuthorizationServerSecurityConfig`, `oidc/`, `token/`, `userinfo/`, `password/`, `HttpSessionConfig` (authorization-server).
> Conceitos em [Identidade e OAuth 2](./fundamentos-identidade-oauth2.md) · o fluxo em [Authorization code e consentimento](./authorization-code-e-consentimento.md).

---

## OIDC não substitui o OAuth2 — ele se apoia nele

O OAuth2 nunca foi um protocolo de login. Ele responde *"o portador pode fazer isto?"*, e nada mais. Quando a indústria começou a usá-lo para autenticar pessoas, cada provedor inventou o seu jeito de dizer quem era o usuário — e o OIDC nasceu para padronizar exatamente essa camada que faltava.

| | Access token | **ID token** |
|---|---|---|
| Responde | *o portador pode fazer isto* | *o portador é esta pessoa* |
| Para quem | as **APIs** (resource servers) | o **client** que pediu o login |
| Formato | JWT (aqui) ou opaco | **sempre** JWT |
| Vai no header `Authorization`? | sim | **não** |

> ⚠️ **O erro clássico do OIDC é confundir os dois**: mandar ID token para a API, ou tentar autorizar com ele. O ID token é um **documento de identidade**, emitido para o aplicativo que fez o login exibir "Olá, John" e montar o perfil. Ele não é credencial de acesso a coisa nenhuma.

Medido — a mesma requisição de token devolve os dois, e o conteúdo deixa a divisão evidente:

| claim | ID TOKEN (para o client) | ACCESS TOKEN (para as APIs) |
|---|---|---|
| `sub` | `019d7764-3b02-7fd5-...` | `019d7764-3b02-7fd5-...` |
| `name` | `John Doe` | — |
| `email` | `john.doe@email.com` | — |
| `type` | `CUSTOMER` | — |
| `createdAt` | `1786767339` | — |
| `scope` | — | `['orders:read', 'openid']` |
| `auth_time` · `azp` · `sid` | presentes | — |
| `exp` | +30 min | **+5 min** |

Repare nos prazos: o ID token vive mais que o access token, e é coerente — ele não abre porta nenhuma. E note o `sid`: o identificador de **sessão**, que é o que torna o logout do próximo capítulo possível.

### `openid` é o escopo que liga a camada

Pedir `openid` é o que transforma um fluxo OAuth2 em OIDC. Sem ele, a resposta de token não traz `id_token`.

E há um detalhe que só aparece testando: **`openid` não aparece na tela de consentimento**.

```
authorize ... &scope=openid orders:read
→ tela de consentimento oferece apenas:  ['orders:read']
→ token concedido:                       scope = "orders:read openid"
```

Faz sentido: consentimento é sobre **permissões em recursos**, e pedir identidade não é uma permissão sobre recurso. Quem consente em identificar-se é quem faz login.

---

## O usuário saiu da memória

Até a fase passada havia um único usuário, declarado em `application.yaml`:

```yaml
spring.security.user:
  name: customer@gmail.com
  password: secret123
```

Agora há uma tabela, uma entidade e um `UserDetailsService`.

### O `AuthUser` é anêmico — e por ora está certo

```java
@Entity
@Table(name = "auth_user")
public class AuthUser extends AbstractAuditableAggregateRoot<AuthUser> {
    @Id private UUID id;
    private String email;
    private String password;
    private String name;
    private boolean enabled;
    @Enumerated(EnumType.STRING) private AuthUserType type;
}
```

Sem construtor público, sem factory, sem uma regra. É o oposto de tudo que o [`ordering`](../02-persistencia/product-catalog-mongo.md) e o `product-catalog` fazem — e ainda assim é a escolha certa **agora**, porque não há operação de negócio sobre usuário: ninguém se cadastra, ninguém troca senha, ninguém é desativado pela aplicação. Os três usuários vêm do `afterMigrate.sql`.

> **Agregado existe para proteger invariante, e invariante só aparece quando há operação que possa violá-la.** Antecipar o modelo rico aqui seria cerimônia: `AuthUser.changePassword()` sem ninguém para chamá-lo é código morto com aparência de arquitetura.
>
> A primeira invariante a aparecer será previsível: **trocar senha exige a senha atual e a nova codificada** — e é nesse dia que o setter deixa de servir.

### `{noop}` e `{bcrypt}` convivendo

```java
Map<String, PasswordEncoder> encoders = Map.of(
    "bcrypt", new BCryptPasswordEncoder(),
    "noop", NoOpPasswordEncoder.getInstance());
return new DelegatingPasswordEncoder("bcrypt", encoders);
```

O seed usa os dois de propósito:

```sql
'{noop}123456'                                  -- john.doe, victoria
'{bcrypt}$2a$10$TYlaa0oLIGnqG5Jdoaa.mePxJD9y...' -- jeff.roman
```

Verificado — os dois autenticam pela mesma tela:

```
{noop}   john.doe@email.com       -> 302 /
{bcrypt} jeff.roman@algashop.com  -> 302 /
senha errada                      -> 302 /login?error
usuario inexistente               -> 302 /login?error
```

> **O prefixo é o que torna a migração de algoritmo possível sem invalidar senha.** Guardar só o hash obriga o sistema a assumir qual algoritmo o produziu; guardar `{alg}hash` deixa cada senha dizer como validar a si mesma. Trocar `bcrypt` por `argon2` amanhã é mudar o default e deixar as antigas seguirem funcionando — recodificadas uma a uma, no próximo login de cada pessoa.

E note que **usuário inexistente e senha errada dão a mesma resposta**. É deliberado: distinguir os dois entregaria a quem sonda uma lista de e-mails cadastrados.

### O `UserDetails` sai sem authorities

```java
return User.withUsername(email)
        .password(user.getPassword())
        .disabled(!user.isEnabled())
        .build();
```

O `AuthUserType` (`MANAGER`, `OPERATOR`, `CUSTOMER`) **não vira `GrantedAuthority`** — ele viaja como claim OIDC. Significa que `hasRole('MANAGER')` dentro do próprio authorization server nunca casaria.

É coerente com o desenho do projeto: [autorização é por escopo, e acontece nos resource servers](./resource-server-e-escopos.md). O `type` é informação de identidade, não permissão. Vale saber que a distinção é essa, e não um esquecimento.

> 🔄 **Isto mudou na Fase 27.** O `AuthUserType` continua não virando authority **aqui** (no `UserDetails` do formulário de login), mas passou a viajar como claim `role` no access token — e a virar `ROLE_MANAGER`/`ROLE_OPERATOR`/`ROLE_CUSTOMER` nos quatro resource servers. A distinção do parágrafo acima segue válida, com um adendo: identidade vira permissão **na fronteira do token**, não dentro do authorization server. Ver [RBAC e controle de acesso](./rbac-e-controle-de-acesso.md).

---

## As duas filter chains

```java
@Bean @Order(1)
public SecurityFilterChain authorizationServerFilterChain(HttpSecurity http) {
    var authorizationServer = new OAuth2AuthorizationServerConfigurer();
    http.securityMatcher(authorizationServer.getEndpointsMatcher())
        .with(authorizationServer, c -> c.oidc(oidc -> oidc
                .logoutEndpoint(l -> l.logoutResponseHandler(oidcLogoutAuthenticationSuccessHandler))
                .userInfoEndpoint(u -> u.userInfoMapper(oidcUserInfoMapper))))
        .authorizeHttpRequests(a -> a.anyRequest().authenticated())
        .exceptionHandling(e -> e.defaultAuthenticationEntryPointFor(
                new LoginUrlAuthenticationEntryPoint("/login"),
                new MediaTypeRequestMatcher(MediaType.TEXT_HTML)));
    return http.build();
}

@Bean @Order(2)
public SecurityFilterChain defaultSecurityFilterChain(HttpSecurity http) {
    http.authorizeHttpRequests(a -> a.anyRequest().authenticated())
        .formLogin(Customizer.withDefaults());
    return http.build();
}
```

**Por que duas.** O Spring avalia as chains na ordem do `@Order` e usa a **primeira** cujo `securityMatcher` casar. A de cima recorta apenas os endpoints do protocolo (`getEndpointsMatcher()`); tudo o mais — a começar pela página de login — cai na de baixo. Inverter a ordem faria o `formLogin` engolir `/oauth2/token`.

**O `LoginUrlAuthenticationEntryPoint` por `MediaTypeRequestMatcher`** é a peça que resolve um conflito real: `/oauth2/authorize` é chamado por **navegador** (quer ser redirecionado ao login) e `/oauth2/token` por **backend** (quer um erro JSON). A negociação de conteúdo decide qual comportamento aplicar.

> É exatamente a mesma mecânica que na fase passada fazia o `curl` levar **401** onde o navegador levava 302: sem `Accept: text/html`, o entry point conclui que quem chama é uma API.

---

## Customizar os tokens

```java
if (isOpenIdToken(tokenType)) {
    context.getClaims().claims(claims -> claims.putAll(oidcUserInfo.getClaims()));
} else if (isAccessToken(tokenType) && (isAuthCodeFlow(...) || isRefreshTokenFlow(...))) {
    context.getClaims().subject(oidcUserInfo.getSubject());
}
```

Duas decisões distintas no mesmo lugar:

**No ID token, acrescenta identidade.** `name`, `email`, `type`, `createdAt` — para o client não precisar chamar `/userinfo` só para exibir um nome. O custo é o tamanho do token e o fato de esses dados ficarem congelados até a próxima renovação.

**No access token, reescreve o `sub`.** Por padrão ele viria com o *username* — o e-mail. Passa a ser o **UUID** do usuário:

```
sub = 019d7764-3b02-7fd5-b0e7-c47c58592857     (era john.doe@email.com)
```

> **Isto é uma mudança de contrato**, e vale registrar como tal: qualquer resource server que tivesse lido `sub` como e-mail passa a receber um UUID. Nenhum lê hoje — mas o motivo de a mudança estar certa é o mesmo que a torna incompatível: **e-mail muda, id não.** Um identificador que o usuário pode alterar não serve para amarrar dados a ele.
>
> E isso finalmente dá destino à pendência mais antiga deste caderno: a auditoria do `product-catalog`, que grava um `UUID` aleatório como autor. O `sub` do access token agora **é** esse UUID.
>
> ✅ **A Fase 25 colheu isso.** O `AuditorAware` dos três serviços que auditam passou a ler o `sub`, e o `GET /api/v1/users/me` do authorization server passou a resolver "quem sou eu" sem receber id nenhum na URL. Ver [Gestão de usuários e auditoria](./gestao-de-usuarios-e-auditoria.md).

Repare no `else if`: o `sub` só é reescrito em `authorization_code` e `refresh_token`. Em `client_credentials` não há pessoa, e o `sub` continua sendo o `client_id` — que é o correto.

---

## `/userinfo`

Se o ID token já traz tudo, para que serve o endpoint? Para quando o client **não** tem o ID token à mão, ou quando quer o dado **atual** em vez do congelado na emissão. O ID token é uma fotografia do momento do login; `/userinfo` é a consulta ao vivo.

```java
var idTokenHolder = authorization.getToken(OidcIdToken.class);
if (idTokenHolder == null) {
    // sem ID token: devolve o mínimo que o OIDC exige
    return OidcUserInfo.builder().claim("sub", principal.getToken().getClaims().get("sub")).build();
}
return new OidcUserInfo(idTokenHolder.getClaims());
```

Verificado, com o access token no header:

```
GET /userinfo  ->  200
  sub  019d7764-3b02-7fd5-b0e7-c47c58592857
  name John Doe    email john.doe@email.com    type CUSTOMER
```

> O ramo de fallback existe porque o padrão do Spring recusa `/userinfo` sem ID Token na autorização. Ele mantém o endpoint utilizável por um portador que só tem access token — devolvendo o mínimo, que é o `sub`.
>
> ⚠️ Como o caminho principal **reaproveita as claims do ID token**, `/userinfo` devolve a fotografia, não o dado vivo. Isso contraria a razão de o endpoint existir, e é uma escolha que só se percebe lendo o mapper.

---

## Logout de verdade

Sair de uma aplicação OIDC significa três coisas diferentes, e o padrão só cobre uma:

| O quê | Quem faz |
|---|---|
| encerrar a **sessão** no authorization server | o RP-initiated logout, do padrão |
| **revogar** as autorizações (tokens vivos) | **decisão da aplicação** |
| invalidar tokens já emitidos nas APIs | ninguém — ver abaixo |

O código desta fase acrescenta a segunda:

```java
private void revokeAuthorizations(Authentication authentication) {
    String email = authentication.getName();
    for (String id : queryService.findAuthorizationIds(email)) {
        OAuth2Authorization a = authorizationService.findById(id);
        if (a != null) authorizationService.remove(a);
    }
}
```

O `OAuth2AuthorizationService` não oferece busca por principal, daí o `JdbcOAuth2AuthorizationQueryService` com um `SELECT id FROM oauth2_authorization WHERE principal_name = ?` — uma consulta de leitura ao lado do serviço da biblioteca, sem substituí-lo.

### Medido

```
ANTES:   autorizacoes de john.doe: 1     sessoes: 2
GET /connect/logout?id_token_hint=...&post_logout_redirect_uri=...
         -> 302  http://algashop-ecommerce:9080/?logout-success
DEPOIS:  autorizacoes de john.doe: 0     sessoes: 1     consent: 1
         sessoes de jeff.roman:    2     (intactas)
```

Três coisas para ler nesse resultado:

**O consentimento sobrevive.** Está certo: sair não é revogar permissão. Da próxima vez que John entrar, não será perguntado de novo — porque ele nunca desfez a decisão, só encerrou a sessão.

**O logout é global por usuário.** `revokeAuthorizations` apaga **todas** as autorizações daquele principal, de **todos** os clients. Sair do app web derruba também o token que outro aplicativo daquele mesmo usuário estivesse usando. É defensável — e não é o comportamento padrão. Precisa ser uma decisão consciente, não um efeito colateral.

**O `post_logout_redirect_uri` precisa estar registrado.** Ele veio da configuração do client; um valor não registrado é recusado, exatamente como o `redirect_uri` da ida.

### E o que o logout **não** faz

```
/userinfo com o access token       ->  401  {"error":"invalid_token"}
refresh com o token revogado       ->  400  {"error":"invalid_grant"}

o mesmo access token, para um resource server:
  assinatura intacta, exp em 248s  ->  ACEITO
```

> **Revogar no authorization server não invalida um JWT já emitido.** O `/userinfo` recusa porque é o próprio servidor resolvendo o token contra o armazenamento — que agora está vazio. Um resource server valida **localmente**, pela assinatura, e não pergunta nada a ninguém: ele aceita o token até o `exp`.
>
> É o preço do JWT, [descrito desde a Fase 20](./authorization-server.md#opaco--jwt-a-escolha-que-decide-o-resto), aparecendo aqui em forma concreta. Os 5 minutos de TTL **são** a janela de logout incompleto. Fechá-la exigiria token opaco com introspecção, ou back-channel logout notificando cada resource server.

---

## Sessão em banco

Com `formLogin`, o authorization server passou a ter **estado de sessão**: é o cookie que sustenta o usuário autenticado entre a tela de login e o `/oauth2/authorize`.

```java
@Configuration
@EnableJdbcHttpSession
public class HttpSessionConfig { }
```

Duas tabelas (`V4`), com schema também ditado pela biblioteca: `SPRING_SESSION` e `SPRING_SESSION_ATTRIBUTES`, com índices por `EXPIRY_TIME` e `PRINCIPAL_NAME` — o segundo é justamente o que permite achar as sessões de uma pessoa.

Medido:

```
john.doe@email.com | expira em 04:45:50 | max_inactive=1800s
jeff.roman@...     | expira em 04:45:51 | max_inactive=1800s
```

**Por que sair da memória:** sessão em memória significa deslogar todo mundo a cada deploy, e impede mais de uma instância — o usuário que autenticou na réplica A chega na B como anônimo. É o mesmo argumento que levou tokens e consentimentos para o banco na fase anterior, aplicado a outro tipo de estado.

> ⚠️ **`spring.session.timeout: 30m` está inerte.** Declarar `@EnableJdbcHttpSession` cria o `SessionRepository` e faz a auto-configuração do Boot recuar — a propriedade deixa de ser lida, e vale o padrão da anotação, que é 1800s. Os dois valores coincidem, então **não há diferença observável hoje**; a configuração parece ativa e não é. Ligar de verdade seria `@EnableJdbcHttpSession(maxInactiveIntervalInSeconds = ...)`, ou remover a anotação e deixar o Boot configurar.
>
> É o terceiro caso do mesmo tipo neste caderno — depois do `reuse-refresh-token` no singular e do `jpa`/`flyway` aninhados no `datasource`.

---

## O achado: a anotação perdida no rename

`PersistenceConfig` foi renomeada para `OAuth2PersistenceConfig` e movida de pacote — e **perdeu o `@Configuration`** no caminho. A suíte acusou:

```
NoSuchBeanDefinitionException: No qualifying bean of type
  'org.springframework.security.oauth2.server.authorization.OAuth2AuthorizationService'
```

A cadeia até a falha vale ser entendida:

1. Sem `@Configuration`, os dois beans de persistência do OAuth2 não são criados.
2. O `OAuth2AuthorizationService` **não é um bean do Spring Authorization Server** — o configurer cria um `InMemory` internamente, como objeto compartilhado da filter chain, e não o publica no contexto.
3. O `OidcRevokeAuthorizationsLogoutHandler` desta fase **injeta** esse tipo. Sem bean, o contexto não sobe.

> **A ironia é o que salva.** Se fosse apenas a persistência, a Fase 23 teria sido **desfeita em silêncio**: tokens e consentimentos de volta à memória, sem erro, sem aviso, e com o sintoma aparecendo semanas depois como "por que todo mundo desloga no deploy?". Foi o código novo, que passou a depender do bean **explicitamente**, que transformou uma regressão invisível numa falha de inicialização.
>
> Generalizando: **depender de um bean por injeção é mais seguro do que depender dele por efeito colateral.** O que é injetado falha alto quando some; o que é apenas "configurado em algum lugar" falha baixo.

---

## Armadilhas

- **Mandar ID token para a API.** Ele identifica; não autoriza.
- **`openid` não aparece no consentimento** — testar consentimento de identidade não funciona por essa via.
- **Logout não invalida JWT já emitido** nos resource servers.
- **`@EnableJdbcHttpSession` torna `spring.session.timeout` inerte.**
- **Rename que perde anotação** — o compilador não ajuda, e a falha pode ser silenciosa.
- **`/userinfo` reaproveita as claims do ID token**, então devolve a fotografia da emissão, não o dado atual.
- **A ordem das filter chains importa**: a do protocolo precisa vir antes da que tem `formLogin`.

---

## Pendências registradas

- ✅ ~~**`@EnableJpaAuditing` não existe**~~ — **ligado na Fase 25**, junto com a resposta para "de onde vem o `AuditorAware<UUID>`": do `sub` deste mesmo ID token. O NPE latente em `OidcUserInfoService.getCreatedAt().toEpochSecond()` deixou de ser latente porque `createdAt` passou a ser preenchido. Ver [Gestão de usuários e auditoria](./gestao-de-usuarios-e-auditoria.md).
- [ ] **O logout é global por usuário**, revogando autorizações de todos os clients. Fazer por client exigiria filtrar pelo `registered_client_id` do `id_token_hint`.
- [ ] **`/userinfo` devolve dados congelados** na emissão do ID token, em vez de consultar o `AuthUser`.
- [ ] **Sem back-channel logout.** Os resource servers não são notificados; um access token vivo continua aceito até expirar.
- [ ] **Senhas `{noop}` no seed** — dois dos três usuários. O terceiro está em `{bcrypt}`, que é o caminho certo.
- ✅ ~~**Não há cadastro, troca de senha nem desativação** pela aplicação. O `AuthUser` é anêmico porque ainda não há operação — quando houver, ele precisa deixar de ser.~~ — **Fase 25**: chegaram cadastro, atualização e anonimização, e o agregado deixou de ser anêmico (fábrica `brandNew`, `anonymize()`, setters validando). Falta ainda a **troca de senha** — e a senha temporária do cadastro não é entregue a ninguém.
- [ ] **`AuthUserType` não vira authority.** Se um dia a autorização precisar de papel (e não só de escopo), o `UserDetails` terá que carregá-lo.
- [ ] **PKCE segue desligado** — pendência herdada da Fase 23.
- [ ] **Typo em `idTokenHoldeer`** no `OidcUserInfoMapper`.

---

## Checklist de revisão

- [ ] O ID token é usado só para identificar, nunca como credencial de API?
- [ ] O `sub` do access token é um identificador **estável**, e não o e-mail?
- [ ] O logout revoga as autorizações, além de encerrar a sessão?
- [ ] O TTL do access token é curto o bastante para ser a janela de logout aceitável?
- [ ] A sessão sobrevive a um restart e é compartilhada entre instâncias?
- [ ] O `post_logout_redirect_uri` está registrado no client?
- [ ] As senhas guardam o **prefixo do algoritmo**?

---

---

## 🔄 O cookie de sessão virou peça de arquitetura (Fase 26)

Nesta fase a sessão descrita acima deixou de ser detalhe interno. Um `CookieSerializer` explícito passou a controlar como ela viaja:

```
Set-Cookie: JSESSIONID=...; Domain=algashop.local; Path=/; HttpOnly; SameSite=Lax
```

O `Domain` é o que permite `auth.algashop.local` e `admin.algashop.local` compartilharem a **mesma** sessão — e foi por isso que os hosts do projeto foram renomeados para debaixo de um pai comum. Sem isso, o *silent refresh* (renovar o access token num iframe escondido, apoiado na sessão) seria estruturalmente impossível.

> Ver [PKCE e clientes públicos](./pkce-e-clientes-publicos.md#o-domínio-comum-não-é-cosmético).

## Referências

- [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html)
- [OIDC RP-Initiated Logout 1.0](https://openid.net/specs/openid-connect-rpinitiated-1_0.html)
- [Spring Authorization Server — OIDC](https://docs.spring.io/spring-authorization-server/reference/guides/how-to-userinfo.html)
- [Spring Session — JDBC](https://docs.spring.io/spring-session/reference/jdbc.html)
- [Authorization code e consentimento](./authorization-code-e-consentimento.md) · [Authorization Server](./authorization-server.md) · [Resource servers e escopos](./resource-server-e-escopos.md)
