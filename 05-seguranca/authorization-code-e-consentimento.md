# Authorization code, consentimento e refresh token

> Todo o OAuth deste projeto até aqui foi **sem usuário**. Este documento é sobre o fluxo com pessoa — e sobre a consequência que vem junto: a partir do momento em que alguém consente, existe estado que precisa sobreviver a um restart.
> Código real: `infrastructure/security/persistence/OAuth2PersistenceConfig.java`, `db/migration/V1__*.sql` e `V2__*.sql`, `application-development-env.yaml` (authorization-server).
> Conceitos em [Identidade e OAuth 2](./fundamentos-identidade-oauth2.md) · o servidor em [Authorization Server](./authorization-server.md) · o outro lado em [OAuth2 client e token](./oauth2-client-e-token.md).

---

## A pergunta que muda

| Grant | A pergunta que ele responde |
|---|---|
| `client_credentials` | *este **serviço** pode ler produto?* |
| `authorization_code` | *esta **pessoa** autorizou este aplicativo a agir por ela?* |

A segunda pergunta traz três coisas que a primeira não tem: **uma tela de login** (alguém precisa provar quem é), **uma tela de consentimento** (essa pessoa precisa dizer o que libera), e **um refresh token** (a sessão dela precisa durar mais que os 5 minutos do access token).

E traz uma quarta, menos óbvia, que é o que fez este servidor ganhar banco:

> **Consentimento é uma decisão do usuário.** Uma decisão do usuário que some no próximo deploy nunca foi uma decisão de verdade — o aplicativo voltaria a pedir permissão do zero, e a pergunta "você já autorizou isto?" não teria resposta.

---

## O fluxo, com o que cada passo protege

```
 navegador                AUTHORIZATION SERVER            client (backend)
    │                                                          │
    │  1. GET /oauth2/authorize?response_type=code&...          │
    ├────────────────────────────▶                             │
    │  2. tela de login                                        │
    │  3. tela de CONSENTIMENTO  ── a pessoa escolhe escopos ── │
    │  4. 302 → redirect_uri?code=...&state=...                │
    ├──────────────────────────────────────────────────────────▶
    │                                                          │
    │                     5. POST /oauth2/token (code + segredo)│
    │                        ◀─────────────────────────────────┤
    │                        access_token + refresh_token       │
```

### Por que existe um *código* no meio

O passo 4 devolve um **código**, não o token. Parece um rodeio e não é: o que volta no passo 4 viaja **pela URL do navegador** — fica no histórico, no header `Referer`, no log do proxy. Um token ali estaria exposto.

O código resolve isso porque sozinho ele não vale nada: para trocá-lo por token é preciso **o segredo do cliente**, que só existe no backend, e a troca acontece de servidor para servidor. O código é de **uso único** e vive 10 minutos.

> É exatamente por isso que o *implicit grant* morreu: ele devolvia o token direto na URL. Ver [fundamentos](./fundamentos-identidade-oauth2.md#o-que-mudou-no-oauth-21).

### O `state`

Vai na ida e volta idêntico — verificado:

```
authorize ... &state=fase23
→ Location: http://algashop-ecommerce:9080/login/oauth2/code/...?code=XuCFffoI...&state=fase23
```

É proteção contra CSRF no callback: o cliente gera um valor aleatório, guarda na sessão, e recusa o retorno cujo `state` não bate. Sem ele, alguém induz a vítima a completar um fluxo que o atacante começou.

---

> 🔄 **Atualizado na Fase 24.** O usuário saiu da memória e foi para o banco (`auth_user`), e o escopo `openid` entrou no client web — o que faz este mesmo fluxo devolver também um **ID token**. Ver [OpenID Connect: identidade, sessão e logout](./openid-connect-e-sessao.md).

## A tela de consentimento

O que a torna diferente do login: o login pergunta **quem é você**; o consentimento pergunta **o que eu deixo este aplicativo fazer por mim**. São perguntas para pessoas diferentes na mesma pessoa.

```
Consent required
algashop-ecommerce-web wants to access your account customer@gmail.com
The following permissions are requested by the above app:
  [ ] customers:write
  [ ] customers:read
```

Ela só aparece por causa de uma linha:

```yaml
require-authorization-consent: true
```

Sem ela o consentimento é **implícito**: autenticou, autorizou tudo que o cliente pedir.

### Granularidade — verificado

Pedi dois escopos e consenti **um**:

```
POST /oauth2/authorize  scope=customers:read     (só um dos dois marcados)
→ 302 com code

POST /oauth2/token
→ {"scope": "customers:read", ...}
```

E o claim do access token confirma:

```json
{ "iss": "http://auth.algashop.local:9000",
  "sub": "customer@gmail.com",
  "aud": "algashop-ecommerce-web",
  "scope": ["customers:read"] }
```

> **O escopo do token é a interseção entre o que o cliente pediu e o que a pessoa concedeu.** O cliente pode estar registrado com oito escopos e receber um. É essa a diferença entre "o app pode pedir" e "eu deixei".

### Acumulativo, e por cliente

A tabela guarda **uma linha por (cliente, pessoa)** — não um histórico:

```sql
PRIMARY KEY (registered_client_id, principal_name)
```

Depois da minha autorização, a linha que já existia dos testes anteriores tinha crescido:

```
algashop-ecommerce-web-client | customer@gmail.com | SCOPE_orders:read,SCOPE_orders:write,SCOPE_customers:read
```

`customers:read` entrou; **`customers:write` não está lá** — foi pedido e recusado. E o efeito prático da acumulação apareceu no passo seguinte: pedir de novo um escopo **já consentido** não mostra tela nenhuma.

```
GET /oauth2/authorize ... &scope=customers:read   →  302 direto, sem consentimento
```

> Não há registro de *quando* cada permissão foi dada, nem trilha de revogação parcial. Para um caderno de estudos, tudo bem; para um sistema que precisa provar consentimento, a chave primária é a limitação.

---

## Refresh token, e a rotação que não estava ligada

O refresh existe para não pedir a senha de novo a cada 5 minutos. E ele é **a credencial mais valiosa em circulação**: quem o tem gera access tokens por 1 hora sem passar por tela nenhuma.

Daí a rotação — cada uso emite um novo e invalida o anterior:

```yaml
reuse-refresh-tokens: false
```

### O achado: o singular é ignorado em silêncio

A configuração dizia `reuse-refresh-token` (**singular**). A propriedade do Spring Boot é `reuse-refresh-tokenS`. Propriedade desconhecida não gera erro nenhum — e o **default é reusar**. Verificado, antes da correção:

```
renovar          → HTTP 200, MESMO refresh token de volta
reusar o antigo  → HTTP 200                                ← deveria falhar
```

Depois de trocar uma letra:

```
renovar          → HTTP 200, refresh DIFERENTE
reusar o antigo  → HTTP 400  {"error":"invalid_grant"}
```

> **A configuração afirmava uma propriedade de segurança que o sistema não tinha.** Não havia erro, aviso, nem sintoma: o fluxo funcionava, o comentário no YAML dizia "sempre revoga um refresh token antigo", e a rotação estava desligada.
>
> É o mesmo mecanismo de outros três achados deste caderno — o nome de cache da Fase 15, o `project(Class)` da Fase 12, o `hasAuthority('SCOPE_...')` da Fase 21. **O que não é verificado em compilação precisa ser verificado por comportamento**, e configuração nunca é verificada em compilação.

### Por que a rotação importa

Sem ela, um refresh vazado vale a hora inteira, em paralelo com o do dono, sem deixar rastro. Com ela, o primeiro dos dois a renovar invalida o outro — e o legítimo sendo recusado é **um sinal detectável** de que houve vazamento.

---

## Os três prazos

```yaml
authorization-code-time-to-live: 10m
access-token-time-to-live: 5m
refresh-token-time-to-live: 1h
```

| | Prazo | O que ele é |
|---|---|---|
| código | 10 min | comprovante de troca, **uso único**, some em segundos na prática |
| access | 5 min | o que circula em toda requisição; curto porque [não se revoga](./authorization-server.md#opaco--jwt-a-escolha-que-decide-o-resto) |
| refresh | 1 h | a duração da **sessão** da pessoa |

Medido: `expires_in: 299` — os 5 minutos, menos o segundo que a requisição levou.

---

## O estado, e por que ele foi para o banco

```java
@Bean
public JdbcOAuth2AuthorizationService authorizationService(...) { ... }

@Bean
public JdbcOAuth2AuthorizationConsentService auth2AuthorizationConsentService(...) { ... }
```

Sem esses dois beans o Spring usa as versões `InMemory`, **que funcionam e não avisam nada**. A troca compra três coisas:

- **O refresh sobrevive ao deploy.** Em memória, cada reinício desloga todo mundo.
- **O consentimento sobrevive.** Que é o argumento de verdade — ver a abertura deste documento.
- **Mais de uma instância passa a ser possível.** Com estado em memória, o refresh emitido por uma réplica é desconhecido pelas outras.

### Uma linha por autorização, não por token

`oauth2_authorization` guarda o código, o access, o refresh e o id token **da mesma concessão** na mesma linha, com metadados e prazos. É o que permite revogar tudo de uma vez: apagar a linha invalida a concessão inteira.

### O schema não se inventa

As duas migrations são cópias fiéis da distribuição do Spring Authorization Server. **Não é um modelo de domínio nosso**: nomes de coluna, tipos e tamanhos são lidos pelo `RowMapper` da biblioteca. Renomear uma coluna ou apertar um `varchar` quebra a persistência em runtime, não em compilação.

> É um caso diferente de tudo o que [`flyway.md`](../02-persistencia/flyway.md) descreve nos outros serviços: aqui a migration não versiona **a nossa** decisão de modelagem — ela versiona a de uma biblioteca.

### O que o banco revela sobre o YAML

```
registered_client_id = algashop-ecommerce-web-client
```

É **a chave do mapa** do YAML, não o `client-id` (`algashop-ecommerce-web`). Isso refina o que [o documento do authorization server](./authorization-server.md) dizia sobre a chave ser "só um apelido interno": ela vira o **identificador persistido** do cliente. Renomeá-la depois órfã todos os consentimentos e autorizações já gravados.

---

## Como reproduzir

```bash
# 1. login (guardando cookie de sessão)
curl -s -c jar -b jar http://auth.algashop.local:9000/login   # extrair o _csrf do HTML
curl -s -c jar -b jar -d "username=customer@gmail.com" -d "password=secret123" -d "_csrf=<...>" \
     http://auth.algashop.local:9000/login

# 2. autorizar, pedindo um escopo AINDA NÃO consentido (senão a tela é pulada)
curl -s -c jar -b jar "http://auth.algashop.local:9000/oauth2/authorize\
?response_type=code&client_id=algashop-ecommerce-web\
&redirect_uri=http%3A%2F%2Falgashop-ecommerce%3A9080%2Flogin%2Foauth2%2Fcode%2Falgashop-ecommerce-web\
&scope=customers%3Aread%20customers%3Awrite&state=teste"

# 3. consentir só parte, e pegar o code do Location
# 4. trocar o code por token
curl -s -u algashop-ecommerce-web:ecommerce123 -d grant_type=authorization_code \
     -d "code=<CODE>" --data-urlencode "redirect_uri=<a mesma>" \
     http://auth.algashop.local:9000/oauth2/token

# 5. renovar, e provar a rotação reusando o antigo -> 400 invalid_grant
curl -s -u algashop-ecommerce-web:ecommerce123 -d grant_type=refresh_token \
     --data-urlencode "refresh_token=<REFRESH>" \
     http://auth.algashop.local:9000/oauth2/token
```

> ⚠️ Com `Accept: */*`, o `/oauth2/authorize` sem sessão responde **401** em vez de redirecionar para o login: o `DelegatingAuthenticationEntryPoint` escolhe por negociação de conteúdo, e sem `Accept: text/html` ele assume que quem chama é uma API e devolve erro OAuth2. É uma confusão fácil ao testar por `curl`.

---

## Armadilhas

- **`reuse-refresh-token` no singular é ignorado** e o default é reusar — a rotação fica desligada sem aviso.
- **Consentimento é por `(cliente, pessoa)` e acumulativo** — pedir menos escopos depois não reduz o que já foi concedido.
- **Escopo já consentido não mostra tela** — testar consentimento exige pedir um escopo novo.
- **A chave do mapa do YAML vira `registered_client_id`** — renomear órfã o que já está gravado.
- **Nunca editar migration já aplicada.** Aconteceu ao escrever este documento: acrescentar um comentário no topo do `.sql` muda o checksum, e o Flyway recusa subir com `FlywayValidateException`. O comentário foi revertido e a explicação veio para cá — que é o lugar dela.
- **`/oauth2/authorize` responde 401 para `Accept: */*`**, e 302 para `text/html`.

---

## Pendências registradas

- [ ] **PKCE desligado** (`require-proof-key: false`) num client `authorization_code`. O OAuth 2.1 o tornou obrigatório, inclusive para client confidencial, e [os fundamentos deste caderno afirmam isso](./fundamentos-identidade-oauth2.md#o-que-mudou-no-oauth-21). Ligar exige gerar `code_verifier`/`code_challenge` no cliente — o que o teste manual não faz hoje.
- [ ] **Tokens em texto puro no banco.** `access_token_value` e `refresh_token_value` são `text`, como o `JdbcOAuth2AuthorizationService` espera. O banco virou um armazém de credenciais portadoras: **quem lê a tabela se passa por qualquer usuário**. Cifrar em repouso, ou tratar esse banco com o mesmo rigor de um cofre.
- [ ] **`logging.level.org.springframework.security: TRACE`** registra credenciais e tokens no log. Excelente para aprender, e não pode chegar a outro ambiente.
- [x] ~~**Um usuário único, em memória, com senha no YAML.**~~ Resolvido na Fase 24: há uma tabela `auth_user`, um `UserDetailsService` sobre ela e um `DelegatingPasswordEncoder`. Continua sem cadastro e sem troca de senha pela aplicação.
- [ ] **Não há tela de gerenciamento de consentimento.** O usuário concede e não tem como revogar; só apagando a linha no banco. O logout da Fase 24 revoga as **autorizações**, mas preserva o consentimento — e está certo: sair não é desfazer permissão.
- [ ] **O app cliente não existe.** O `redirect_uri` aponta para `http://algashop-ecommerce:9080`, que não está no repositório — o fluxo é exercitável até o redirect, e ali termina.
- [ ] **`docker-env` e `production-env` continuam vazios**, e agora sem datasource o servidor nem sobe nesses perfis.
- [ ] **`init-user-db.sh` só roda com o volume do Postgres vazio.** Quem já tinha o banco criado não ganha o `authorization_server`, e a falha na subida não diz isso.
- [ ] **O client `algashop-ecommerce-m2m` carrega `customers:write`** — plausível para auto-cadastro (criar conta antes de existir usuário para autenticar), e ainda assim é autoridade de escrita num cliente sem pessoa por trás.

---

## Checklist de revisão

- [ ] O `state` é gerado, guardado e conferido no retorno?
- [ ] O código é trocado **de servidor para servidor**, com o segredo do cliente?
- [ ] `require-authorization-consent` está ligado para clientes que agem em nome de alguém?
- [ ] A rotação de refresh está **ligada de fato** — e não só escrita no YAML?
- [ ] O prazo do refresh corresponde à duração de sessão que se quer?
- [ ] O estado (tokens e consentimentos) sobrevive a um restart?
- [ ] Existe caminho para o usuário **revogar** o que consentiu?

---

---

## 🔄 O mesmo fluxo, sem segredo (Fase 26)

Tudo descrito acima vale para um cliente **confidencial** — que se autentica com `client_secret` na troca do código. A Fase 26 acrescentou um cliente **público**, que roda no navegador e não tem segredo nenhum para apresentar.

O fluxo é o mesmo, com duas diferenças:

- no `/oauth2/authorize` viaja um `code_challenge` — o hash SHA-256 de um segredo descartável que o cliente acabou de sortear;
- no `/oauth2/token` viaja o `code_verifier` original, no lugar onde o `client_secret` iria.

O servidor recalcula o hash e compara. É o **PKCE**, e ele substitui a pergunta "quem é você?" pela pergunta "foi você que começou isto?".

E o mesmo endpoint ganhou um uso novo: com `prompt=none`, o `/oauth2/authorize` renova o token **sem tela**, apoiado na sessão — é o *silent refresh*, que existe porque um cliente público não deveria guardar refresh token.

> Documento completo: [PKCE e clientes públicos](./pkce-e-clientes-publicos.md).

## Referências

- [RFC 6749 §4.1 — Authorization Code Grant](https://datatracker.ietf.org/doc/html/rfc6749#section-4.1)
- [RFC 6749 §6 — Refreshing an Access Token](https://datatracker.ietf.org/doc/html/rfc6749#section-6)
- [OAuth 2.0 Security Best Current Practice — refresh token rotation](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics)
- [Spring Authorization Server — Core Model / Components](https://docs.spring.io/spring-authorization-server/reference/core-model-components.html)
- [Identidade e OAuth 2](./fundamentos-identidade-oauth2.md) · [Authorization Server](./authorization-server.md) · [OAuth2 client e token](./oauth2-client-e-token.md)
