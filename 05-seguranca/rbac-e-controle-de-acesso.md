# RBAC: papéis, permissões e o fluxo guiado de uma requisição

Até a Fase 26 a autorização do AlgaShop respondia **uma** pergunta: *este token tem o escopo?*

Escopo é o teto do que um **client** pode pedir. Ele é cego para quem está usando: dois funcionários no mesmo aplicativo carregavam exatamente os mesmos poderes. Esta fase acrescenta a segunda pergunta — *quem é esta pessoa?* — e com ela quatro camadas de controle que se empilham.

Este documento percorre **a jornada de uma requisição** por essas camadas, na ordem em que elas realmente acontecem.

---

## Escopo × papel: por que os dois

| | Escopo | Papel (*role*) |
|---|---|---|
| Responde | *este aplicativo pode fazer isto?* | *esta pessoa é quem?* |
| Pertence a | o **client** | o **usuário** |
| Onde é declarado | YAML do client | `auth_user.type` |
| Vira | `SCOPE_users:write` | `ROLE_MANAGER` |

Nenhum substitui o outro, e a confusão entre eles é o erro clássico de RBAC.

> **Escopo delega; papel identifica.** Um escopo diz "a SPA de admin pode escrever usuário" — não diz *qual* pessoa dentro dela pode. Um papel diz "Victoria é MANAGER" — não diz por qual aplicativo ela está entrando.

E é a combinação que fica interessante: o mesmo `MANAGER` tem poderes diferentes conforme o client, e o mesmo client entrega poderes diferentes conforme o papel.

---

# O fluxo guiado

```
                  ┌─────────────────────────────────────────────────────────────┐
   PESSOA ───────▶│ 1. /oauth2/authorize                                        │
                  │    "este papel pode este client?"   auth_user_type_client_allowed
                  │    "este papel pode estes escopos?" auth_user_type_client_scope
                  │    ✗ -> access_denied / invalid_scope  (NENHUM código é emitido)
                  └────────────────────────┬────────────────────────────────────┘
                                           │ ✓ code
                  ┌────────────────────────▼────────────────────────────────────┐
                  │ 2. /oauth2/token — JwtTokenCustomizer                        │
                  │    sub = UUID do usuário   +   role = MANAGER                │
                  └────────────────────────┬────────────────────────────────────┘
                                           │ access token
                  ┌────────────────────────▼────────────────────────────────────┐
                  │ 3. no resource server — JwtGrantedAuthoritiesDelegating…     │
                  │    scope -> SCOPE_*      role -> ROLE_*                      │
                  └────────────────────────┬────────────────────────────────────┘
                                           │ Authentication com authorities
                  ┌────────────────────────▼────────────────────────────────────┐
                  │ 4. a API decide                                             │
                  │    @CanWriteUsers      -> escopo                            │
                  │    canRegisterUserOf…  -> papel                             │
                  │    id do token == id do recurso -> dono                     │
                  └─────────────────────────────────────────────────────────────┘
```

---

## Passo 1 — o `/oauth2/authorize` filtra antes de existir código

A decisão mais importante desta fase é **onde** a autorização passou a acontecer. Ela deixou de ser só uma checagem na porta da API e virou **critério de emissão**: quem não pode, não recebe token.

Duas tabelas sustentam isso:

```sql
auth_user_type_client_allowed (auth_user_type, client_id)
-- "este papel pode ABRIR este aplicativo?"

auth_user_type_client_scope   (auth_user_type, client_id, scope)
-- "abrindo, o que ele pode LEVAR?"
```

Ambas com chave primária composta pela linha inteira: elas não têm identidade própria, **são** o próprio fato. E ambas com o mesmo default — **ausência de linha significa negado**. Um client novo no YAML não fica disponível para ninguém até que alguém decida, explicitamente, quem pode usá-lo.

O `AuthUserClientAccessPolicyValidator` é quem pergunta, e ele roda **depois do login e antes do código**:

```java
DEFAULT_REDIRECT_URI_VALIDATOR      // protocolo: a redirect_uri está registrada?
  .andThen(DEFAULT_SCOPE_VALIDATOR) // protocolo: os escopos existem no client?
  .andThen(clientAccessPolicyValidator)  // negócio: este papel pode?
```

> ⚠️ O `setAuthenticationValidator()` **substitui** o validador padrão, não acrescenta. Plugar a política direto ali desligaria, em silêncio, a checagem de redirect URI — a que impede o código de autorização de ser entregue no servidor de um atacante. Daí o `andThen`: primeiro o protocolo, depois o negócio.

### Por que negar aqui é melhor que negar na API

- a pessoa descobre na tela, no momento do login, e não navegando depois;
- **nenhum token com poder indevido chega a ser assinado** — e token assinado não se revoga (a lição de JWT da Fase 20 cobrando a conta);
- a regra fica num lugar só, em vez de replicada em cada resource server.

### Verificado

Mesma requisição, três pessoas:

```
CUSTOMER  -> admin-web : access_denied  "The authenticated user type is not allowed to authorize this client."
MANAGER   -> admin-web : code emitido
OPERATOR  -> admin-web : code emitido
```

E a simetria, que é o que mostra que a regra é uma política e não um remendo:

```
MANAGER   -> ecommerce-web : access_denied
```

O gerente não entra na loja pelo client do cliente. A separação vale nos dois sentidos.

---

## Passo 2 — o token nasce com papel

```java
// JwtTokenCustomizer
context.getClaims().subject(oidcUserInfo.getSubject());   // sub = UUID, não e-mail
context.getClaims().claim("role", role);                  // MANAGER | OPERATOR | CUSTOMER
```

Duas escritas, e só duas. O ID token recebe as claims de identidade (nome, e-mail, tipo); o **access token** recebe apenas `sub` e `role`. Mandar identidade para o access token seria vazamento — ele circula em toda requisição a toda API, e nenhuma delas precisa saber o e-mail de quem chama.

**Só em fluxo com pessoa.** `authorization_code` e `refresh_token` passam por aqui; `client_credentials` não. Verificado:

```
MANAGER  -> sub=019d7764-3b02-7be2-9112-039fda30e965  role=MANAGER
máquina  -> sub=algashop-test                          role=None
```

> ⚠️ O `refresh_token` **precisa** estar na lista. Sem ele, renovar o token devolveria um access token sem `role` — a pessoa perderia os poderes na primeira renovação, cinco minutos depois de logar, sem erro nenhum no caminho.

---

## Passo 3 — `role` vira `ROLE_*`

O `JwtGrantedAuthoritiesConverter` padrão do Spring lê o claim `scope` e prefixa cada valor com `SCOPE_`. É só isso que ele faz. O converter do projeto **delega** para ele e soma uma authority:

```java
HashSet<GrantedAuthority> authorities = new HashSet<>(scopeGrantedAuthorities.convert(jwt));
String role = jwt.getClaimAsString("role");
if (StringUtils.isNotBlank(role)) {
    authorities.add(new SimpleGrantedAuthority("ROLE_" + role));
}
```

```
scope: "users:read users:write"  ->  SCOPE_users:read, SCOPE_users:write
role:  "MANAGER"                 ->  ROLE_MANAGER
```

**O prefixo não é enfeite.** `hasRole('MANAGER')` monta a string `"ROLE_MANAGER"` por conta própria e compara. Gravar a authority como `"MANAGER"` faria `hasRole('MANAGER')` falhar em silêncio — e `hasAuthority('MANAGER')` passar. Duas formas de escrever a mesma intenção que **não** são equivalentes, e nenhuma verificada em compilação.

E há uma ligação frágil que vale conhecer:

```java
@Bean
public JwtAuthenticationConverter jwtAuthenticationConverter(JwtGrantedAuthoritiesDelegatingConverter c) { ... }
```

O `.jwt(withDefaults())` da `SecurityConfig` não recebe o converter como parâmetro — ele **procura** um bean desse tipo. Esquecer a `@Configuration` deixaria o converter inerte: contexto sobe, testes passam, token continua sendo aceito, e o `role` simplesmente nunca vira authority. Toda regra por papel passaria a negar todo mundo, com um 403 sem explicação como único sintoma.

> Mesma família do `@Configuration` perdido num rename (Fase 24) e do `@TestConfiguration` que ninguém importava (Fase 26).

---

## Passo 4 — a API decide, em três níveis

```java
@CanWriteUsers                       // 1. escopo   -> @PreAuthorize("hasAuthority('SCOPE_users:write')")
public AuthUserOutput create(...) {
    securityCheck.canRegisterUserOfType(input.getType());   // 2. papel
}

// 3. dono
if (!securityCheck.isCustomer() || !getAuthenticatedUserId().equals(order.getCustomer().getId())) { ... }
```

Os três **se somam**, e a ordem em que falham é observável. Verificado, criando usuário:

```
MANAGER   criando MANAGER   -> 201
MANAGER   criando OPERATOR  -> 201
MANAGER   criando CUSTOMER  -> 403  "Cannot register user of type CUSTOMER"

OPERATOR  criando qualquer  -> 403  "You do not have permission to access this resource"

máquina   criando MANAGER   -> 403  "Cannot register user of type MANAGER"
máquina   criando CUSTOMER  -> 201
```

**Repare que as mensagens do OPERATOR são diferentes das outras.** Não é inconsistência: ele nem chega à regra de negócio. `users:write` não está na lista dele em `auth_user_type_client_scope`, então o token nunca carrega o escopo, e o `@PreAuthorize` barra antes. Duas barreiras para a mesma regra, em camadas diferentes — e a mensagem denuncia qual delas atuou.

E a assimetria que mais ensina: **quem cadastra cliente é a máquina.** `canRegisterUserOfType(CUSTOMER)` exige `isMachineAuthenticated()`. Cliente se cadastra pela loja (`client_credentials` do `algashop-ecommerce-m2m`), não por alguém do back-office criando conta em nome dele. É o raro caso em que "ser uma máquina" aparece como **permissão**, e não como restrição.

### Edição: três perguntas, não uma

```
MANAGER editando OPERATOR (outro)     -> 200
MANAGER editando o PRÓPRIO registro   -> 200
MANAGER promovendo OPERATOR->MANAGER  -> 200
MANAGER editando um CUSTOMER          -> 403  "Cannot edit user of type CUSTOMER"
```

`canEditUser` responde *"posso mexer neste registro?"*; `canChangeUserType` responde *"posso mexer neste campo?"*. São perguntas diferentes — um OPERATOR pode editar o próprio nome e não pode promover-se.

O **próprio registro sempre pode ser editado**, qualquer que seja o papel. É a mesma ideia do `/me`: o id não vem da requisição, vem do token, então não há como pedir para editar o de outro por engano.

---

## O id compartilhado é a chave que atravessa os serviços

A regra "só o dono acessa" se resume a uma comparação:

```java
securityCheck.getAuthenticatedUserId().equals(order.getCustomer().getId())
```

Ela só significa alguma coisa porque os três serviços concordam sobre o id:

```
authorization-server   auth_user.id        = 6e148bd5-47f6-4022-b9da-07cfaa294f7a
algashop-ordering      customer.id         = 6e148bd5-47f6-4022-b9da-07cfaa294f7a
algashop-billing       invoice.customer_id = 6e148bd5-47f6-4022-b9da-07cfaa294f7a
```

**A identidade é a chave estrangeira que atravessa a fronteira dos serviços.** Sem essa igualdade, a comparação nunca daria verdadeiro e a regra seria decorativa — cada `CUSTOMER` veria uma lista vazia e nunca saberia por quê.

É o pagamento da decisão da Fase 24 (`sub` virou UUID) e da Fase 25 (auditoria pelo `sub`), agora cobrada em três bancos diferentes.

---

## Filtrar em vez de negar

Duas formas de proteger uma consulta, e o `OrderQueryService` usa as duas:

```java
public Page<OrderSummaryOutput> filter(OrderFilter filter) {
    if (securityCheck.isCustomer()) {
        filter.setCustomerId(securityCheck.getAuthenticatedUserId());  // FILTRA
    }
    return forObtainingOrder.filter(filter);
}

public OrderDetailOutput findById(String orderId) {
    OrderDetailOutput order = forObtainingOrder.findById(orderId);
    if (!canAccess(order)) {
        throw new AccessDeniedException(...);                          // NEGA
    }
    return order;
}
```

| | Quando usar | O que o cliente vê |
|---|---|---|
| **Filtrar** | listagem | 200, com menos dados. Pedir o `customerId` de outro é ignorado, não recusado |
| **Negar** | recurso único | 403 |

A escolha não é estilística. Numa listagem, negar exigiria que o cliente adivinhasse o próprio filtro; e responder 403 a "liste pedidos" seria estranho quando existe uma resposta correta — a lista dele. Num recurso único não há meio-termo: ou é seu, ou não é.

> Repare no efeito colateral do 403 em `findById`: ele **confirma que o pedido existe**. Devolver 404 para recurso alheio esconderia isso. É uma escolha em aberto, registrada nas pendências.

---

## O achado: permitido em um lugar, bloqueado no outro

A prova revelou algo que nenhum teste pegava. O `CUSTOMER` estava corretamente listado em `auth_user_type_client_allowed` para o client da loja — e **sem uma única linha** em `auth_user_type_client_scope`:

```
auth_user_type | client_id              | escopos
---------------+------------------------+---------
MANAGER        | algashop-admin-web     |      12
OPERATOR       | algashop-admin-web     |      10
                                             ↑ nenhuma linha para CUSTOMER

auth_user_type | client_id
---------------+------------------------
CUSTOMER       | algashop-ecommerce-web    ← permitido a entrar...
```

Resultado, para qualquer escopo pedido:

```
CUSTOMER -> ecommerce-web : invalid_scope
```

**"Pode entrar e não pode fazer nada."** O fluxo inteiro da loja estava quebrado — e o erro aponta para o *pedido* (`invalid_scope`), não para a *configuração que falta*, que é o que torna esse tipo de bug caro de achar.

> 🔄 **Fase 28:** a tela de consentimento que o usuário vê depois disto passou a ser própria — e ela só é alcançada por quem **já passou** por estas duas tabelas. Consentir é a última porta, não a primeira. Ver [Telas e formulários de login](./telas-e-formularios-de-login.md).

> **As duas tabelas se preenchem em conjunto.** `allowed` responde "pode abrir?"; `scope` responde "pode levar o quê?". Uma sem a outra produz um usuário que autentica e não autoriza — o pior dos dois mundos, porque parece funcionar até o último passo.

A lição mais geral: **política espalhada em duas tabelas precisa de uma verificação que olhe as duas juntas.** Nenhum teste unitário pegaria isso, porque cada tabela, isolada, está correta.

---

## O segundo achado: a renomeação que não compila

`canAccessOwnProfile()` virou `isCustomer()` na porta do `ordering`. O `TestSecurityCheckConfig` implementava a antiga:

```
error: <anonymous TestSecurityCheckConfig$1> is not abstract and does not override
       abstract method isCustomer() in SecurityCheckApplicationService
```

Este é o **bom** tipo de quebra — o compilador acusou, na hora, no lugar certo. Compare com o mesmo tipo de mudança feita por string (nome de bean em SpEL, escopo em `hasAuthority`), que passa pela compilação e nega todo mundo em runtime.

> **Trocar um método de interface é seguro; trocar o texto dentro de um `@PreAuthorize` não é.** A diferença não é de risco intrínseco — é de quem verifica.

Atrás dele veio o efeito colateral esperado: `BuyNow` e `Checkout` passaram a exigir **ser o dono**, e 13 testes que rodavam sem papel nenhum caíram em `AccessDenied`. A correção não foi afrouxar a regra — foi dar aos testes a identidade que a produção tem:

```java
TestAuthentications.authenticateAsCustomer(CustomerTestDataBuilder.DEFAULT_CUSTOMER_ID.value());
```

E `TestAuthentications` monta o `Jwt` com os mesmos claims que o `JwtTokenCustomizer` escreve e o passa pelo **mesmo converter** que roda em produção. Mockar a porta faria o teste afirmar apenas "quando eu digo que é um CUSTOMER, o serviço trata como CUSTOMER" — uma tautologia.

Um deles mudou de significado, e para melhor:

```java
// era: 422 "cliente não encontrado"
// virou: 403
void shouldReturnForbiddenWhenOrderingForAnotherCustomer()
```

A verificação de dono passou à frente da consulta ao banco, e com isso a API **deixou de confirmar quais ids de cliente existem** para quem não tem direito a eles. Mesma razão pela qual o login responde igual para senha errada e usuário inexistente (Fase 24).

---

## A matriz de quem pode o quê

| | MANAGER | OPERATOR | CUSTOMER | máquina |
|---|---|---|---|---|
| Client permitido | `admin-web` | `admin-web` | `ecommerce-web` | — (sem `/authorize`) |
| Escopos no client | 12 | 10 | 10 | os do client |
| Criar MANAGER/OPERATOR | ✅ | ❌ | ❌ | ❌ |
| Criar CUSTOMER | ❌ | ❌ | ❌ | ✅ |
| Editar o próprio registro | ✅ | ✅ | ✅ | ❌ |
| Editar registro de outro | ✅ *(≠ CUSTOMER)* | ❌ | ❌ | ❌ |
| Promover/rebaixar | ✅ *(MANAGER ↔ OPERATOR)* | ❌ | ❌ | ❌ |
| Ver pedidos de todos | ✅ | ✅ | ❌ *(só os seus)* | — |
| Fazer pedido | ❌ | ❌ | ✅ *(só para si)* | ❌ |

Duas leituras que valem ser notadas:

**Ninguém "cria um CUSTOMER" pelo painel** — nem o MANAGER. Cliente nasce pela loja.

**Nem MANAGER nem OPERATOR fazem pedido.** `verifyCanOrderFor` exige `isCustomer()`. É uma decisão, não um esquecimento — mas fecha a porta para "comprar em nome do cliente" por telefone, que costuma ser um requisito real de back-office. Registrado nas pendências.

---

## Armadilhas

- **`setAuthenticationValidator()` substitui, não acrescenta** — encadeie os validadores padrão ou perca a validação de redirect URI.
- **O prefixo `ROLE_`** — `hasRole('X')` e `hasAuthority('X')` não são a mesma coisa.
- **O bean `JwtAuthenticationConverter`** é procurado, não injetado; sem ele o converter é inerte e tudo passa a negar.
- **`refresh_token` fora do `JwtTokenCustomizer`** faria o papel sumir na primeira renovação.
- **Ausência de linha = negado**, nas duas tabelas — e uma sem a outra produz "entra e não faz nada".
- **Migration aplicada não se edita**, nem para acrescentar comentário: o checksum muda e o Flyway recusa subir. *(Aconteceu nesta fase, ao comentar V5 e V6.)*
- **O `CHECK` do enum está em dois lugares** — no `AuthUserType` e no DDL. Papel novo no Java sem migration correspondente falha em runtime.

## Pendências registradas

- [ ] **MANAGER e OPERATOR não conseguem fazer pedido para um cliente** — back-office por telefone não tem caminho.
- [ ] **`findById` de pedido alheio responde 403**, confirmando que o pedido existe. 404 esconderia.
- [ ] **A política vive no seed**, não numa tela — mudar quem pode o quê exige SQL.
- [ ] **`isMachineAuthenticated()` continua sendo heurística** (`aud` × `sub`), agora com mais peso: ela decide quem cadastra cliente.
- [ ] **Não há teste que verifique as duas tabelas em conjunto** — foi assim que o buraco do CUSTOMER passou.
- [ ] **`AuthUserType` não vira authority no próprio authorization server** para o `/login`; só chega como claim no token.

## Checklist de revisão

- [ ] Papel novo? Então: enum, `CHECK` do DDL, linhas em **allowed** e em **scope**.
- [ ] Client novo? Ninguém o usa até haver linha em `auth_user_type_client_allowed`.
- [ ] Escopo novo no client? Ele não chega a papel nenhum até entrar em `auth_user_type_client_scope`.
- [ ] A regra é de escopo ou de papel? Escopo vai na meta-anotação; papel, no `SecurityCheckApplicationService`.
- [ ] É listagem ou recurso único? Filtre a primeira, negue o segundo.
- [ ] O teste autentica com papel de verdade, ou mocka a porta e prova uma tautologia?

## Referências

- [NIST RBAC](https://csrc.nist.gov/projects/role-based-access-control)
- [Spring Security — Method Security](https://docs.spring.io/spring-security/reference/servlet/authorization/method-security.html)
- [Spring Authorization Server — Authorization endpoint](https://docs.spring.io/spring-authorization-server/reference/configuration-model.html)
- [Resource servers e escopos](./resource-server-e-escopos.md) · [Gestão de usuários e auditoria](./gestao-de-usuarios-e-auditoria.md) · [PKCE e clientes públicos](./pkce-e-clientes-publicos.md)
