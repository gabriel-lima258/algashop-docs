# PKCE e clientes públicos

Até a Fase 25, todo cliente do AlgaShop guardava um segredo. Esta fase acrescenta o primeiro que **não pode guardar nada**: uma SPA de administração, que roda inteira dentro do navegador.

A primeira metade deste documento explica o **PKCE isoladamente** — ele existe fora deste projeto e vale por si. A segunda mostra o que foi construído em cima dele.

---

# Parte 1 — PKCE, sozinho

## O problema

No fluxo `authorization_code`, o authorization server não devolve o token direto. Devolve um **código**, e o código volta pelo **navegador** — numa URL:

```
http://app.exemplo/callback?code=RRIl1Kd6sK0Eq&state=abc
```

Esse trajeto é o ponto fraco. A URL passa por lugares que a aplicação não controla: histórico do navegador, log de servidor intermediário, extensão instalada, e — no caso de app mobile — o registro de *custom URL scheme* do sistema operacional, que **outro aplicativo pode reivindicar**. Quem capturar o código nesse caminho pode tentar trocá-lo por um token.

Para um cliente confidencial isso não basta: a troca do código exige também o `client_secret`, que o atacante não tem. O código sozinho não vale nada.

> **O `client_secret` é o que impede o código roubado de virar token.**

E é exatamente isso que falta num cliente público. Uma SPA, um app mobile, um CLI — o "segredo" deles está no bundle JavaScript, no APK, no binário. Está a um *view-source* de distância. Um segredo que todo mundo tem não é segredo: é uma string.

Sem `client_secret`, o código interceptado é suficiente. **O PKCE devolve ao fluxo a proteção que o segredo dava, sem precisar de segredo.**

## O mecanismo, em quatro passos

*Proof Key for Code Exchange* — RFC 7636. A ideia cabe numa frase: **em vez de um segredo permanente que o cliente guarda, um segredo descartável que ele inventa na hora.**

```
1. o cliente sorteia          code_verifier   = "C41wK3Bg9BJ8L4OwkrX4YyOOavqSAirueKBLqiyB-Cw"
                                               (43-128 caracteres aleatórios, um por requisição)

2. e deriva                   code_challenge  = BASE64URL( SHA256( code_verifier ) )
                                              = "0uV0LsMxNWO0Z972KinknHUEjnD_sdrxogKhWpReI7Q"

3. GET /oauth2/authorize?...&code_challenge=0uV0Ls...&code_challenge_method=S256
                              └── só o RESUMO viaja pelo navegador; o servidor o guarda junto ao código

4. POST /oauth2/token         code=...&code_verifier=C41wK3...
                              └── agora o ORIGINAL viaja, e pelo canal direto (back-channel)

   o servidor recalcula SHA256(code_verifier) e compara com o challenge que guardou
```

Quem interceptou a URL do passo 3 tem o *challenge* — um hash. Para trocar o código ele precisaria do *verifier*, que nunca passou por ali e existe só na memória do cliente que iniciou o fluxo.

> **O PKCE amarra quem PEDIU o código a quem o TROCA.**

## Por que `S256` e não `plain`

A RFC também admite `code_challenge_method=plain`, em que o challenge **é** o verifier. Isso destrói o mecanismo: quem intercepta a URL do passo 3 já tem tudo de que precisa para o passo 4.

`plain` só existe para dispositivos incapazes de calcular SHA-256, e a própria RFC diz que o cliente **deve** usar `S256` quando puder. Na prática: se você viu `plain` numa configuração, é bug.

## PKCE não é autenticação de cliente

É a confusão mais comum, e vale separar:

| | O que prova |
|---|---|
| `client_secret` | **identidade** — "eu sou o cliente `algashop-admin-web`" |
| PKCE | **continuidade** — "quem está trocando este código é quem o pediu" |

O PKCE não diz *quem* é o cliente. Um atacante pode iniciar um fluxo próprio, com verifier próprio, se passando pelo `client_id` da sua SPA — e vai conseguir, porque `client_id` é público por definição. O que ele **não** consegue é sequestrar um fluxo que outra pessoa começou.

É por isso que redirect URIs registradas continuam sendo essenciais: elas é que impedem o código de ser entregue no lugar errado. PKCE e redirect URI resolvem metades diferentes do mesmo problema.

## Cliente público × confidencial

```yaml
# público — a SPA
client-authentication-methods:
  - none          # não se autentica: não tem com o quê
require-proof-key: true

# confidencial — o serviço
client-authentication-methods:
  - client_secret_basic
```

`none` é uma declaração, não uma omissão: diz ao servidor "este cliente não vai apresentar credencial, e está certo assim". Na troca do código, o `client_id` vai no corpo em vez de num header `Authorization: Basic`.

## O que o OAuth 2.1 mudou

O OAuth 2.1 consolidou o que a prática já tinha decidido:

- **PKCE obrigatório para todos** os clientes que usam `authorization_code` — inclusive os confidenciais. A justificativa: mesmo com segredo, o código interceptado é uma superfície que não precisa existir.
- **O grant `implicit` morreu.** Ele existia justamente para clientes públicos: devolvia o token direto na URL, sem código e sem troca. Com PKCE, o `authorization_code` passou a servir clientes públicos com segurança — e o `implicit`, que punha token no histórico do navegador, perdeu a razão de existir.
- **Password grant também saiu**, por motivo relacionado: nenhum cliente deveria ver a senha do usuário.

---

# Parte 2 — O que o AlgaShop construiu em cima

## O quinto cliente

```yaml
algashop-admin-web-client:
  registration:
    client-id: algashop-admin-web
    authorization-grant-types:
      - authorization_code            # e SÓ ele — repare no que não está aqui
    client-authentication-methods:
      - none
    redirect-uris:
      - http://admin.algashop.local:4200
      - http://admin.algashop.local:4200/silent-refresh.html
  require-authorization-consent: false
  require-proof-key: true
  token:
    access-token-time-to-live: 5m
    authorization-code-time-to-live: 2m
```

Três decisões que valem ser lidas com atenção:

**Não há `refresh_token`.** Não é esquecimento. Um refresh token é uma credencial de longa duração, e guardá-la no navegador reproduz exatamente o problema do `client_secret` — em `localStorage` ela é legível por qualquer XSS; em memória ela some ao recarregar a página. A resposta desta fase é outra: **silent refresh** (adiante).

**Não há consentimento.** A tela de consentimento existe para cliente de **terceiro** — quando uma aplicação que não é sua pede acesso aos seus dados. Num cliente próprio, feito pela mesma organização que opera o servidor, pedir consentimento é teatro: o usuário está autorizando a empresa a acessar dados que ela já tem. Compare com o `algashop-ecommerce-web`, que também é próprio e **exige** consentimento — uma inconsistência que vale registrar.

**Quem pode usar este client não está aqui.** Desde a Fase 27, abrir o `admin-web` depende do **papel** do usuário — `MANAGER` e `OPERATOR` entram, `CUSTOMER` leva `access_denied` no `/oauth2/authorize`. E os escopos que cada um leva também variam. Nada disso está no YAML: está em duas tabelas. Ver [RBAC e controle de acesso](./rbac-e-controle-de-acesso.md).

**O código dura 2 minutos, não 10.** Código de autorização é usado imediatamente, uma vez só. Prazo curto reduz a janela de qualquer interceptação.

## A prova — quebrando o PKCE

O fluxo feliz prova pouco: um `200` só diz que algo funcionou. O que prova o mecanismo é **quebrá-lo**. Tudo abaixo é saída real, contra o servidor rodando.

**1. O caminho feliz — cliente público, sem segredo nenhum:**

```
/authorize -> 302  code=SckoqYs1dkXJjtSE...
/token     -> 200  sub=019d7764-3b02-7be2-9112-039fda30e965
                   aud=algashop-admin-web  scope=['openid', 'users:read']
           chaves da resposta: ['access_token', 'expires_in', 'id_token', 'scope', 'token_type']
```

Nenhum `client_secret` foi enviado. E **não há `refresh_token` na resposta** — a configuração diz a verdade.

**2. O mesmo código, com outro `code_verifier`:**

```
400  {'error': 'invalid_grant'}
```

Este é *o* teste do PKCE. O código é válido, não expirou, não foi usado — e mesmo assim não vira token, porque quem o apresenta não prova ter começado o fluxo.

**3. Sem `code_verifier` nenhum na troca:**

```
400  {'error': 'invalid_client'}
```

**Repare no erro: `invalid_client`, não `invalid_grant`.** Não é detalhe. Para um cliente público, o `code_verifier` **é** a autenticação do cliente — é a única coisa que ele apresenta. Omiti-lo não é "faltou um parâmetro do grant"; é "você não se identificou". O servidor está dizendo, com precisão, que PKCE ocupa o lugar que o `client_secret` ocuparia.

**4. Sem `code_challenge` no `/authorize`, num cliente com `require-proof-key: true`:**

```
302  error=invalid_request
     error_description=OAuth 2.0 Parameter: code_challenge
```

A recusa vem **no início** do fluxo, não no fim — o servidor não emite um código que já sabe que não poderá ser trocado com segurança.

**5. Código de uma requisição, verifier de outra:**

```
400  {'error': 'invalid_grant'}
```

O par é por requisição. Ter *um* verifier válido não serve; tem que ser **o** verifier daquele código.

**6. O cliente confidencial, sem PKCE, continua passando:**

`algashop-ecommerce-web` tem `require-proof-key: false` e o `/authorize` sem `code_challenge` segue em frente (cai na tela de consentimento, que ele exige). É a pendência de PKCE aberta desde a Fase 23, agora medindo-se contra um cliente que faz certo ao lado.

---

## Silent refresh

Sem refresh token, o access token de 5 minutos expira e a SPA precisa de outro **sem mandar a pessoa logar de novo**. A solução não inventa nada: refaz o `authorization_code` do começo, invisivelmente.

```
        SPA (admin.algashop.local:4200)
              │
              │  cria um <iframe> escondido apontando para
              │  auth.algashop.local:9000/oauth2/authorize?...&prompt=none
              ▼
        ┌──────────────────────────────────────────────┐
        │  o navegador manda o COOKIE DE SESSÃO junto   │  ← a peça que faz tudo funcionar
        │  o AS reconhece a sessão, não mostra tela     │
        │  e responde 302 para .../silent-refresh.html  │
        │  com um code novo na URL                      │
        └──────────────────────────────────────────────┘
              │
              │  a página dentro do iframe repassa o code para a SPA
              ▼
        a SPA troca o code por um access token novo (com PKCE)
```

`prompt=none` é o parâmetro do OpenID Connect que diz: *"faça isto sem interagir com o usuário; se não der, devolva erro em vez de mostrar tela"*. Verificado, com sessão válida:

```
/authorize?prompt=none -> 302  code=gY8TnYC187uolOyP...   (nenhuma tela)
/token                 -> 200  access_token novo emitido
```

O usuário não viu nada. A sessão no authorization server continua sendo a fonte da verdade; o access token curto é só uma projeção dela.

### O achado: `prompt=none` sem sessão não faz o que promete

A especificação do OIDC é clara: sem usuário autenticado, `prompt=none` deve devolver **`error=login_required`** para a `redirect_uri`, para que a aplicação saiba que precisa mandar a pessoa logar. Foi o que testei — e não foi o que aconteceu:

```
/authorize?prompt=none (sem cookie de sessão)
  -> 302  Location: http://auth.algashop.local:9000/login
          code=None   error=None
```

**Redirecionou para a tela de login.** O log do servidor mostra por quê:

```
AuthenticatedAuthorizationManager   : Checking authorization on GET /oauth2/authorize?...&prompt=none
ExceptionTranslationFilter          : Sending AnonymousAuthenticationToken ... to authentication entry point
DelegatingAuthenticationEntryPoint  : Match found! Executing LoginUrlAuthenticationEntryPoint
```

O `/oauth2/authorize` exige usuário autenticado **na filter chain**. Sem sessão, o `ExceptionTranslationFilter` intercepta e o entry point manda para `/login` — e o endpoint de autorização, onde o `prompt=none` seria lido, **nunca é alcançado**. O parâmetro não é ignorado por bug de implementação: ele nem chega a ser examinado.

É a mesma lição de ordem de camadas da Fase 21, quando `@Valid` respondia 400 antes de o `@PreAuthorize` responder 403:

> **A camada de fora decide primeiro, e ela não conhece as regras da de dentro.**

E a consequência prática é ruim justamente por ser silenciosa. Dentro do iframe escondido, a SPA esperava `login_required` para reagir; recebe uma **página de login renderizada dentro do iframe** — que o `frame-ancestors 'self'` até permite carregar. Nenhuma mensagem chega, nada falha visivelmente, e a renovação simplesmente não acontece.

Repare que a resposta muda com o `Accept`, por causa do entry point negociado por conteúdo da Fase 24:

| `Accept` | resposta |
|---|---|
| `text/html,...` | 302 para `/login` |
| `*/*` | 302 para `/login` |
| `application/json` | **401** |

Um iframe navega como HTML. Fica registrado como pendência: para o `prompt=none` funcionar como a spec manda, o `/oauth2/authorize` precisaria de tratamento próprio antes do entry point genérico.

---

## O domínio comum não é cosmético

Nesta fase os nomes mudaram:

```
algashop-authorization-server:9000   →   auth.algashop.local:9000
algashop-ecommerce:9080              →   algashop.local:9080
(novo)                               →   admin.algashop.local:4200
```

Parece arrumação. Não é — é o que **torna o silent refresh possível**.

O cookie de sessão emitido pelo `CookieConfig`:

```
Set-Cookie: JSESSIONID=...; Domain=algashop.local; Path=/; HttpOnly; SameSite=Lax
```

Duas propriedades aí fazem o trabalho:

**`Domain=algashop.local`** — o cookie vale para o domínio e todos os subdomínios. `auth.algashop.local` o emite, e ele acompanha requisições para qualquer `*.algashop.local`. Um cookie só pode ser marcado para um domínio do qual o emissor é subdomínio: `auth.algashop.local` pode declarar `algashop.local`, mas **nunca** poderia declarar algo que abrangesse `algashop-ecommerce` — os nomes antigos não tinham pai comum nenhum. Era estruturalmente impossível compartilhar sessão entre eles.

**`SameSite=Lax`** — impede o cookie de ir em requisições *cross-site*. E é aqui que "site" precisa ser distinguido de "origem":

| conceito | definição | `auth.algashop.local` × `admin.algashop.local` |
|---|---|---|
| **same-origin** | esquema + host + porta idênticos | **não** — hosts diferentes |
| **same-site** | mesmo domínio registrável (eTLD+1) | **sim** — ambos sob `algashop.local` |

O iframe do silent refresh é *cross-origin* mas *same-site*. Por isso o `Lax` deixa o cookie passar, e por isso o CORS é necessário (ele olha origem) enquanto o `SameSite` não atrapalha (ele olha site).

Se a SPA vivesse em, digamos, `admin-algashop.com`, seria cross-site: o `Lax` bloquearia o cookie no iframe, e a saída seria `SameSite=None; Secure` — que exige HTTPS e vem sendo restringido pelos navegadores. **A escolha dos nomes é a escolha da arquitetura.**

> Os três nomes precisam estar no `/etc/hosts` da máquina de quem desenvolve — a lista está em `etc/hostnames/hostnames` do meta. Ver [`ambiente-local.md`](../04-infraestrutura/ambiente-local.md).

---

## CORS e CSP são permissões diferentes

Confundir as duas é o erro clássico de quem integra SPA com authorization server. Elas respondem a perguntas opostas:

| | pergunta | quem pergunta |
|---|---|---|
| **CORS** | a SPA pode **chamar** o AS por `fetch`/XHR? | o navegador, antes da chamada |
| **CSP `frame-ancestors`** | o AS pode ser **embutido** num iframe da SPA? | o navegador, ao renderizar |

CORS é sobre a SPA alcançar o servidor. `frame-ancestors` é sobre o servidor deixar-se enquadrar. Uma não substitui a outra, e o silent refresh precisa das duas.

```java
cors.setAllowCredentials(true);   // sem isto o cookie de sessão não acompanha a chamada
```

`allowCredentials(true)` tem um preço embutido: com ele o navegador **proíbe** `Access-Control-Allow-Origin: *`. Origem tem que ser listada explicitamente — que é o motivo de `algashop.security.cors.allowedOrigins` existir. Verificado:

```
Origin: http://admin.algashop.local:4200   -> 200  Access-Control-Allow-Origin: http://admin.algashop.local:4200
                                                   Access-Control-Allow-Credentials: true
Origin: http://evil.example.com            -> 403
```

E o CSP que a chain do protocolo devolve:

```
Content-Security-Policy: frame-ancestors 'self' http://admin.algashop.local:4200 http://auth.algashop.local:9000
```

O `'self'` na lista é o que permite a página de login ser renderizada dentro do próprio iframe — o efeito colateral do achado acima.

---

## O achado, de novo: propriedade obrigatória derruba o perfil de teste

`AlgaShopSecurityProperties` é `@Validated` com `@NotNull` em `cors`, `csp` e `cookie`. As propriedades foram declaradas no `application-development-env.yaml`; o grupo de perfis `test` é `base + test-env`. Resultado, nos 8 testes que sobem contexto:

```
Property: algashop.security.cors     Reason: must not be null
Property: algashop.security.csp      Reason: must not be null
Property: algashop.security.cookie   Reason: must not be null
Caused by: BindValidationException: Binding validation errors on algashop.security
```

**É a quarta vez** que uma propriedade nova no `development-env` quebra o perfil de teste — Fases 19, 21, 22 e agora 26. O padrão já é previsível o bastante para virar checklist: *acrescentou propriedade obrigatória? o `test-env` precisa dela também.*

Mas há uma diferença que vale notar, e é a favor deste caso. Compare a mensagem acima com a da Fase 25:

| | mensagem | onde aponta |
|---|---|---|
| Fase 25 (bean faltando) | `NoSuchBeanDefinitionException: RegisteredClientRepository` | quatro beans depois da causa |
| Fase 26 (property faltando) | `BindValidationException ... algashop.security.cors: must not be null` | exatamente na causa, com o nome da propriedade |

`@ConfigurationProperties` + `@Validated` falha **cedo e no lugar certo**. Foi por isso que a correção declarou os valores no `test-env` em vez de dar defaults à classe: o default faria o servidor subir em produção sem CSP, em silêncio — trocando um erro barulhento por um buraco calado.

---

## Armadilhas

- **`plain` como `code_challenge_method`** anula o PKCE. Se aparecer, é bug.
- **PKCE não autentica o cliente** — protege a continuidade do fluxo, não a identidade de quem o iniciou.
- **`allowCredentials(true)` proíbe `Allow-Origin: *`** — origem tem que ser explícita.
- **CORS ≠ `frame-ancestors`** — permitir uma não permite a outra.
- **Cookie com `Domain` só funciona sob domínio-pai comum** — nomes sem pai comum não compartilham sessão, e nenhuma configuração conserta isso.
- **`prompt=none` pode nunca ser lido** se a filter chain barrar antes.
- **Refresh token em SPA** — `localStorage` é legível por XSS; memória some no reload. Silent refresh evita a escolha.

## Pendências registradas

- [ ] **`prompt=none` sem sessão redireciona para `/login`** em vez de devolver `login_required`. Dentro do iframe, a SPA fica esperando uma mensagem que não chega.
- [ ] **PKCE segue desligado no `algashop-ecommerce-web`** — o OAuth 2.1 o exige também de clientes confidenciais.
- [ ] **Consentimento inconsistente entre clientes próprios** — o admin não pede, o e-commerce pede.
- [ ] **`secure: false` no cookie** — correto em HTTP local, inaceitável fora dele. Com HTTPS vira `true`, e aí `SameSite=None` passa a ser opção.
- [ ] **`http://` nas redirect URIs** — aceitável só em desenvolvimento.
- [ ] **Sem back-channel logout** — encerrar a sessão no AS não avisa a SPA, que segue com o access token até o `exp`.

## Checklist de revisão

- [ ] O cliente é público? Então `client-authentication-methods: none` **e** `require-proof-key: true`.
- [ ] O `code_challenge_method` é `S256`?
- [ ] O verifier é sorteado **por requisição**, e não reaproveitado?
- [ ] As redirect URIs incluem a página do silent refresh?
- [ ] O cookie de sessão tem `Domain` num pai comum a todos os hosts envolvidos?
- [ ] A origem da SPA está em `allowedOrigins` **e** em `frame-ancestors`?
- [ ] A propriedade nova de configuração foi declarada também no `test-env`?

## Referências

- [RFC 7636 — Proof Key for Code Exchange](https://datatracker.ietf.org/doc/html/rfc7636)
- [OAuth 2.1 (draft)](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1)
- [OAuth 2.0 for Browser-Based Apps](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-browser-based-apps)
- [OpenID Connect Core — Authentication Request (`prompt`)](https://openid.net/specs/openid-connect-core-1_0.html#AuthRequest)
- [MDN — SameSite cookies](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie/SameSite)
- [Authorization code e consentimento](./authorization-code-e-consentimento.md) · [OpenID Connect e sessão](./openid-connect-e-sessao.md) · [Authorization Server](./authorization-server.md)
