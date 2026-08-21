# As telas do authorization server: Thymeleaf, login próprio e consentimento

Desde a Fase 23 o AlgaShop tem fluxo com pessoa — e essa pessoa via **a tela padrão do Spring Security**. Ela funciona, e denuncia exatamente o que uma empresa não quer denunciar: qual biblioteca guarda as senhas.

Esta fase troca isso por páginas próprias. Mas o assunto não é HTML: é **onde a apresentação encosta na segurança**. Uma tela de login não é uma página que envia dados — é uma página que precisa falar a língua exata que um filtro espera, e cujo endereço precisa estar liberado no mesmo filter chain que ela existe para atender.

---

## O contrato invisível do `formLogin`

Isto é o que o `UsernamePasswordAuthenticationFilter` procura, e nada disso está escrito em código Java do projeto:

| O que | Onde vive | O que acontece se errar |
|---|---|---|
| campo `username` | atributo `name` no HTML | senha correta devolve `/login?error` |
| campo `password` | atributo `name` no HTML | idem |
| `POST /login` | `th:action` do formulário | 404, ou o POST cai noutra rota |
| campo `_csrf` | **ninguém escreve** — o Thymeleaf injeta | **403** em todo login |

> **Quatro strings sustentam a autenticação inteira, e o compilador não vê nenhuma delas.** É a mesma família do nome de bean em SpEL (Fase 25) e do escopo em `hasAuthority` (Fase 21) — agora dentro de um arquivo que nem é Java.

O erro é especialmente cruel porque **não parece um erro**: a tela aparece, o botão funciona, a requisição vai, e a resposta é "credenciais inválidas". Tudo indica senha errada.

### O `_csrf` que ninguém escreveu

Nenhum dos templates contém um campo `_csrf`. Ele aparece assim mesmo:

```
$ curl -s http://auth.algashop.local:9000/login | grep -oE 'name="(username|password|_csrf)"'
  name="_csrf"
  name="password"
  name="username"
```

Quem o coloca é o Thymeleaf: o processador de `th:action` detecta o `CsrfToken` na requisição e injeta o input escondido. **Trocar `th:action="@{/login}"` por `action="/login"` remove o campo** — e o login passa a devolver 403 para todo mundo, sem nenhuma mudança aparente no HTML escrito.

É o tipo de dependência que só se descobre quebrando, e é por isso que esta fase deixou um teste em cima dela.

---

## `loginPage()` liga duas coisas ao mesmo tempo

```java
http.authorizeHttpRequests(authorize -> authorize
                .requestMatchers("/login", "/forgot-password",
                        "/css/**", "/js/**", "/img/**", "/favicon.ico").permitAll()
                .anyRequest().authenticated())
        .formLogin(form -> form.loginPage("/login")
                .defaultSuccessUrl(properties.getDefaultRedirectUri())
                .permitAll());
```

`loginPage("/login")` diz duas coisas de uma vez:

1. **onde renderizar** o formulário, e
2. **para onde redirecionar** quem tentar acessar algo protegido sem sessão.

E é justamente por (2) que a rota **precisa** estar liberada. Sem `permitAll`, o pedido de `/login` cai em `anyRequest().authenticated()`, que redireciona para… `/login`. **Loop de redirecionamento** — o navegador acusa `ERR_TOO_MANY_REDIRECTS` e não há uma linha de log dizendo o porquê.

### Estático também precisa ser público

`/css/**`, `/img/**`, `/favicon.ico`. Se ficarem fora, a página de login carrega — **sem estilo nenhum**. E aqui está a armadilha: o navegador não reclama de um `<link>` que devolve 302 para a tela de login. A página simplesmente aparece feia, e a causa está no filter chain, não no CSS.

Verificado:

```
/login           -> 200  text/html;charset=UTF-8
/css/main.css    -> 200  text/css
/img/logo.png    -> 200  image/png
/favicon.ico     -> 200  image/x-icon
```

> ⚠️ Os arquivos estáticos precisam **entrar no commit**. Eles nasceram fora do controle de versão nesta fase, e um clone sem eles renderiza as três telas sem estilo — com todos os testes passando.

---

## `defaultSuccessUrl` × `SavedRequest`: o destino depende de como você chegou

```java
.defaultSuccessUrl(properties.getDefaultRedirectUri())   // http://algashop.local:9080
```

Parece dizer "depois do login, vá para a loja". Diz menos que isso: `defaultSuccessUrl` sem `alwaysUse=true` só age quando **não existe requisição salva**.

| Como chegou ao login | Para onde vai depois |
|---|---|
| digitou `/login` direto | `http://algashop.local:9080` (o default) |
| foi mandado por `/oauth2/authorize` | **de volta ao `/oauth2/authorize`** — o `SavedRequest` vence |

A segunda linha é o que faz o fluxo OAuth2 funcionar: o `ExceptionTranslationFilter` guarda a requisição original antes de mandar para o login, e o `SavedRequestAwareAuthenticationSuccessHandler` a retoma. Se `defaultSuccessUrl` vencesse sempre, todo login no meio de uma autorização terminaria na home da loja e o código nunca seria emitido.

Verificado nas duas situações:

```
POST /login (direto)  -> 302  Location: http://algashop.local:9080
GET  /      (logado)  -> 302  Location: http://algashop.local:9080/
```

> O destino é **valor fixo de configuração**, não parâmetro de requisição — por isso apontar para fora do domínio não é *open redirect*. Se um dia vier da URL, precisa de lista de permitidos.

---

## O cookie que a Fase 26 preparou

```
Set-Cookie: JSESSIONID=...; Domain=algashop.local; Path=/; HttpOnly; SameSite=Lax
```

O login por formulário é quem **cria** a sessão que o silent refresh da Fase 26 depois consome, no iframe. As duas fases se encontram aqui: o `Domain` no cookie é o que permite `auth.algashop.local` e `admin.algashop.local` compartilharem-na. Ver [PKCE e clientes públicos](./pkce-e-clientes-publicos.md#o-domínio-comum-não-é-cosmético).

---

## A tela de consentimento agora é nossa

```java
.authorizationEndpoint(endpoint -> endpoint
        .authenticationProviders(this::customizeAuthenticationProviders)
        .consentPage("/oauth2/consent"))
```

Uma linha tira do Spring a tela padrão. O que ela **não** faz é escrever a página: o `AuthorizationConsentController` recebe os parâmetros que o servidor passa, separa escopos já aprovados dos que faltam aprovar, e traduz cada um em frase legível.

```
GET /oauth2/authorize?...&scope=openid orders:read shopping-carts:write
  -> 302 /oauth2/consent?scope=orders:read openid shopping-carts:write&client_id=...&state=xKGS-hvOm...

    orders:read              Read orders.
    shopping-carts:write     Create and modify shopping carts.
```

Três coisas para notar nessa saída:

**`openid` não aparece na lista.** O controller o pula de propósito — consentimento é sobre permissão em recurso, e pedir identidade não é uma. Mesma decisão documentada na Fase 24, agora visível no código que desenha a tela.

**O `state` do formulário não é o seu.** `state=fase28` foi o que o client mandou; o que chega à tela é `xKGS-hvOmSQwkm3l...`, gerado pelo servidor. São coisas diferentes: o do client protege contra CSRF no redirecionamento; o interno amarra esta tela àquela requisição de autorização específica. Repassá-lo intacto no formulário é o que permite ao `/oauth2/authorize` reconhecer o consentimento como resposta ao pedido certo.

**`products:read` virou "Read products."** É a única parte disto que é sobre pessoas: escopo é vocabulário de máquina, e ninguém consente com o que não entende. A tabela de descrições mora no controller — e vale saber que ela pode **desatualizar em silêncio**: escopo novo sem entrada na tabela aparece como *"UNKNOWN SCOPE."*

### O que a Fase 27 já filtrou antes disto

A tela de consentimento só é alcançada por quem **já passou** pela política de papel. Um `CUSTOMER` tentando o `admin-web` nunca chega aqui — leva `access_denied` no `/oauth2/authorize`. E um `OPERATOR` pedindo `users:write` leva `invalid_scope` antes de a tela existir.

> **Consentir é a última porta, não a primeira.** O usuário só decide sobre o que a política já permitiu que ele pedisse. Ver [RBAC e controle de acesso](./rbac-e-controle-de-acesso.md).

---

## Logout em duas etapas

```
GET  /logout  -> 200, página de confirmação com um formulário
POST /logout  -> o LogoutFilter encerra a sessão
```

O `LogoutController` só renderiza; quem executa é o filtro do Spring, no POST. A separação não é cerimônia: **logout por GET é vulnerável a CSRF** — bastaria um `<img src="http://auth.algashop.local:9000/logout">` numa página qualquer para deslogar quem a visitasse. Exigindo POST com `_csrf`, o ataque some.

E há duas saídas de logout convivendo, com públicos diferentes:

| Rota | Quem usa | O que faz |
|---|---|---|
| `POST /logout` | a pessoa, pela tela | encerra a **sessão** no authorization server |
| `GET /connect/logout` | o client, via OIDC | RP-initiated logout: encerra a sessão **e revoga** as autorizações |

Ver [OpenID Connect: identidade, sessão e logout](./openid-connect-e-sessao.md).

---

## A suíte que trava o que o compilador não vê

Oito testes novos, e o mais importante deles não olha comportamento nenhum — olha o HTML:

```java
assertThat(html)
        .contains("name=\"username\"")
        .contains("name=\"password\"")
        .contains("name=\"_csrf\"")     // injetado pelo Thymeleaf, não escrito por nós
        .contains("action=\"/login\"");
```

**A matriz tem dentes — e a prova revelou algo melhor que o esperado.** Trocando `name="username"` por `name="user"` no template:

```
LoginPageIT > loginPageShouldCarryTheFieldsTheFilterExpects()  FAILED
LoginPageIT > shouldAuthenticateWithValidCredentials()         PASSED   ← !
```

Exatamente um teste vermelho. Mas repare em qual **passou**: o teste que faz login de verdade continuou verde com a tela quebrada, porque `formLogin()` do `spring-security-test` **monta o POST sozinho**, com os nomes corretos — ele nunca lê a página.

> **Um teste de login que não lê o HTML não testa o formulário.** Ele testa o filtro, que já funcionava. A tela pode estar completamente quebrada para o usuário e essa suíte inteira ficar verde.

É a lição mais transferível desta fase: quando o defeito mora na *ligação* entre duas peças, testar cada peça isolada não encontra nada.

O restante da suíte fixa quem é público e quem não é:

```
GET /login           anônimo -> 200
GET /css/main.css    anônimo -> 200
GET /oauth2/consent  anônimo -> 302 /login
GET /                anônimo -> 302 /login
POST /login  senha errada     -> 302 /login?error
POST /login  usuário inexistente -> 302 /login?error   (mesma resposta, de propósito)
```

---

## O achado: a quinta vez pelo mesmo motivo

`defaultRedirectUri` entrou como `@NotBlank` no `AlgaShopSecurityProperties` e foi declarada só no `application-development-env.yaml`. O grupo de perfis `test` é `base + test-env`:

```
Property: algashop.security.defaultRedirectUri
Reason: must not be null
```

**Fases 19, 21, 22, 26 e 28.** O padrão parou de ser surpresa e virou item de checklist:

> **Propriedade obrigatória nova no `development-env`? O `test-env` precisa dela também.**

O lado bom continua valendo: `@ConfigurationProperties` + `@Validated` falha **cedo e no lugar certo**, nomeando a propriedade. Compare com a Fase 25, em que um bean faltando apareceu quatro beans adiante da causa.

---

## Armadilhas

- **Renomear um campo do formulário** não quebra o build e produz "credenciais inválidas" para a senha certa.
- **`action` no lugar de `th:action`** remove o `_csrf` e devolve 403 em todo login.
- **`loginPage()` sem `permitAll`** produz loop de redirecionamento.
- **Estático fora do `permitAll`** — a página carrega sem estilo, e nada reclama.
- **Estático fora do commit** — mesmo sintoma, em outra máquina.
- **`defaultSuccessUrl` não é o destino sempre** — o `SavedRequest` tem precedência.
- **Escopo novo sem descrição** aparece como *"UNKNOWN SCOPE."* na tela de consentimento.
- **CSS por caminho literal** (`href="/css/main.css"`) em vez de `th:href="@{...}"` quebra sob context path.

## Pendências registradas

- ✅ ~~**O fluxo de recuperação de senha é só casca.**~~ — **implementado na Fase 30**: as três telas ganharam rota no `PublicPasswordController`, mais uma quarta (`forgot-password-message`) que faltava. Ver [Verificação de e-mail e troca de senha](./verificacao-de-email-e-troca-de-senha.md).
- [ ] **Font Awesome vem de um CDN** (`cdnjs.cloudflare.com`) nas seis páginas — dependência de terceiro numa tela de login: sem rede os ícones somem, e é superfície de cadeia de suprimentos num lugar sensível.
- [ ] **Não há teste da tela de consentimento** renderizando com escopos — só do redirecionamento para o login.
- [ ] **A tabela de descrições de escopo vive no controller**, e desatualiza em silêncio.
- [ ] **Sem internacionalização** — as telas são em inglês, as mensagens de erro do domínio em português.

## Checklist de revisão

- [ ] O formulário usa `th:action`, e não `action`?
- [ ] Os campos se chamam exatamente `username` e `password`?
- [ ] A rota da tela está no `permitAll`, junto com `/css/**`, `/img/**` e o favicon?
- [ ] Os arquivos estáticos estão versionados?
- [ ] O logout exige `POST`?
- [ ] Propriedade de configuração nova foi declarada também no `test-env`?
- [ ] Há um teste que lê o **HTML**, e não só um que exercita o filtro?

## Referências

- [Spring Security — Form Login](https://docs.spring.io/spring-security/reference/servlet/authentication/passwords/form.html)
- [Spring Security — CSRF](https://docs.spring.io/spring-security/reference/servlet/exploits/csrf.html)
- [Spring Authorization Server — Custom consent page](https://docs.spring.io/spring-authorization-server/reference/guides/how-to-custom-consent-page.html)
- [Thymeleaf + Spring Security](https://www.thymeleaf.org/doc/tutorials/3.1/thymeleafspring.html)
- [Authorization code e consentimento](./authorization-code-e-consentimento.md) · [OpenID Connect e sessão](./openid-connect-e-sessao.md) · [RBAC e controle de acesso](./rbac-e-controle-de-acesso.md)
