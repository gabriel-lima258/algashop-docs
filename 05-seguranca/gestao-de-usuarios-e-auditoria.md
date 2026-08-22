# Gestão de usuários, `/me` e auditoria com identidade real

A Fase 24 pôs o UUID do usuário no `sub` do token. Esta fase **usa** esse `sub` — e ao usá-lo fecha a pendência mais antiga do caderno, aberta desde a Fase 8.

Duas frentes que se encontram:

1. Uma **API de usuários** no authorization server — listar com filtros, cadastrar, atualizar e anonimizar.
2. A resposta para **"quem está chamando?"**, extraída do token e disponível para a aplicação inteira, nos quatro serviços.

---

## A pendência de dezessete fases

Desde que a auditoria existe, o autor de cada linha era isto:

```java
@Bean
public AuditorAware<UUID> auditorProvider() {
    return () -> Optional.of(UUID.randomUUID());   // ← um autor novo a cada escrita
}
```

Um valor sempre presente, sempre diferente, que nunca identificou ninguém. As colunas `created_by_user_id` e `last_modified_by_user_id` estavam preenchidas e eram **inúteis** — pior que vazias, porque pareciam dados.

Não era descuido: até a Fase 21 nenhum serviço sabia quem estava do outro lado. O placeholder era honesto quanto ao que se podia saber. O que mudou é que agora dá para saber:

```java
@Bean
public AuditorAware<UUID> auditorProvider(SecurityCheckApplicationService securityCheck) {
    return () -> {
        if (!securityCheck.isAuthenticated() || securityCheck.isMachineAuthenticated()) {
            return Optional.empty();
        }
        return Optional.of(securityCheck.getAuthenticatedUserId());
    };
}
```

E a prova, com o servidor de pé:

```
sub do token da Victoria:  019d7764-3b02-7be2-9112-039fda30e965

 email                        | created_by_user_id
------------------------------+--------------------------------------
 carlos.auditado@algashop.com | 019d7764-3b02-7be2-9112-039fda30e965
 maquina.criou@algashop.com   |                                      ← criado por client_credentials
```

> **`Optional.empty()` para máquina é uma decisão, não uma lacuna.** "Criado por ninguém" é mais honesto que "criado por um UUID aleatório". Quando `client_credentials` cria o registro, não existe pessoa responsável — e a coluna nula diz isso. Se um dia for preciso saber *qual sistema* criou, isso é outra coluna, não este campo.

---

## A porta: `SecurityCheckApplicationService`

A camada de aplicação não importa `SecurityContextHolder`. Ela conversa com uma interface de quatro perguntas:

```java
public interface SecurityCheckApplicationService {
    UUID getAuthenticatedUserId();
    boolean isAuthenticated();
    boolean isMachineAuthenticated();
    boolean canAccessOwnProfile();
}
```

A implementação (`OAuth2SecurityCheckApplicationServiceImpl`) fica na infraestrutura, porque é ela que sabe que existe JWT, `SecurityContextHolder` e Spring Security. É a mesma separação que o `ordering` faz entre `CustomersPersistenceProvider` e `Customers`: a aplicação pede o que precisa; quem sabe *como* fica na borda.

**As quatro cópias.** O arquivo é idêntico byte a byte nos quatro serviços, mudando só o `package`. É deliberado — microsserviço independente não compartilha jar de aplicação — e tem preço: **um bug precisa ser corrigido quatro vezes**. Nesta fase isso aconteceu, com dois bugs de uma vez (adiante).

---

## Token de usuário × token de máquina

A distinção é feita por heurística, não por um claim que diga o que é:

```java
List<String> audience = jwt.getAudience();
return audience != null && audience.contains(jwt.getSubject());
```

O raciocínio: em `client_credentials`, o `sub` do token **é o próprio `client_id`**, que também aparece na audiência. Em `authorization_code`, o `sub` é o UUID do usuário e a audiência é o client. Verificado nos dois:

| | `sub` | `aud` | leitura |
|---|---|---|---|
| máquina | `algashop-test` | `algashop-test` | `aud` contém `sub` → máquina |
| usuário | `019d7764-3b02-…` | `algashop-ecommerce-web` | não contém → pessoa |

Funciona, e é frágil: depende de uma **coincidência estrutural** de como o Spring Authorization Server monta o token, não de uma afirmação explícita. Um client cujo `client_id` fosse igual a algum `aud` de token de usuário quebraria a leitura. O caminho robusto seria um claim próprio no token (`token_type: machine`), acrescentado no `OAuth2TokenCustomizer`.

---

## `/me`: o id que o cliente não escolhe

Duas rotas devolvem um usuário:

```
GET /api/v1/users/{userId}     ← o cliente diz de quem quer
GET /api/v1/users/me           ← o servidor decide, lendo o token
```

A diferença não é de conforto, é de **superfície de ataque**. A primeira forma exige que alguém, em algum lugar, verifique se o `{userId}` pedido é o do chamador — e essa verificação é fácil de esquecer, de esquecer em *um* endpoint entre vinte, ou de escrever errado. É a vulnerabilidade mais comum de API REST (IDOR, *insecure direct object reference*).

Em `/me` não há o que verificar: o id **não vem da requisição**. Vem do token, que foi assinado por quem emitiu.

```java
@GetMapping
@CanAccessOwnProfile
public AuthUserOutput findMe() {
    return authUserQueryService.findById(securityCheck.getAuthenticatedUserId());
}
```

E a proteção é sobre *ter perfil*, não sobre escopo:

```
GET /users/me  com token de usuário  → 200  {"id":"019d7764-…","email":"victoria.garcia@algashop.com"}
GET /users/me  com token de máquina  → 403  "You do not have permission to access this resource"
```

Máquina não tem perfil. A recusa não é falta de permissão — é ausência de sujeito.

### O `@PreAuthorize` que chama um bean pelo nome

```java
@Service("securityCheck")                                    // ← o nome
public class OAuth2SecurityCheckApplicationServiceImpl { ... }

@PreAuthorize("@securityCheck.canAccessOwnProfile()")        // ← e a referência a ele
public @interface CanAccessOwnProfile {}
```

O SpEL resolve `@securityCheck` **pelo nome do bean, em runtime**. Renomear a classe sem tocar no `@Service("...")` não quebra nada; mudar a string quebra a autorização inteira **sem erro de compilação** — e a IDE não acusa, porque para ela aquilo é texto.

É a mesma família do `@Component("cache")` (Fase 17), do `hasAuthority('SCOPE_…')` (Fase 21) e do `reuse-refresh-token` no singular (Fase 23):

> **O que não é verificado em compilação precisa ser verificado por comportamento.**

Ao lado dele convivem os escopos, que são verificação de permissão e não de sujeito:

```java
@PreAuthorize("hasAuthority('SCOPE_users:read')")   public @interface CanReadUsers {}
@PreAuthorize("hasAuthority('SCOPE_users:write')")  public @interface CanWriteUsers {}
```

Provado por token:

| token | rota | resultado |
|---|---|---|
| sem token | `GET /users` | **401** |
| `users:read` | `GET /users` | **200** |
| `users:read` | `POST /users` | **403** |
| `products:read` | `GET /users` | **403** |
| token inválido | `GET /users` | **401** |

---

## O `AuthUser` deixou de ser anêmico

Na Fase 24 ele era uma linha da tabela com getters. Agora tem fábrica, invariantes e uma operação de negócio:

```java
public static AuthUser brandNew(String email, String name, AuthUserType type, String passwordHash) {
    // id UUIDv7, enabled = true, setters validando
}

public void anonymize() {
    setName("Anonymized User");
    setEmail("anonymized-" + getId() + "@deleted.local");
    setEnabled(false);
}
```

**`DELETE` não apaga — anonimiza**, no mesmo espírito do arquivamento de cliente do `ordering`:

```
DELETE /api/v1/users/01a0112b-746b-7e6d-8ceb-8c8691c126b3  →  204

 name            | email                                                         | enabled | created_by_user_id
-----------------+---------------------------------------------------------------+---------+--------------------------------------
 Anonymized User | anonymized-01a0112b-746b-7e6d-8ceb-8c8691c126b3@deleted.local | f       | 019d7764-3b02-7be2-9112-039fda30e965
```

A linha continua lá. Tinha de continuar: o id dela pode estar gravado como autor em qualquer registro auditado de qualquer serviço, e apagá-la deixaria essas referências penduradas. Repare que o `created_by_user_id` **sobreviveu à anonimização** — quem criou continua registrado; quem *é* o usuário é que foi apagado.

E a regra que o agregado protege, alcançável pelo `PUT`:

```
PUT /users/{id de um CUSTOMER}  {"type":"MANAGER"}
  → 422  "Cannot change type of a CUSTOMER user"
```

---

## 🔄 Quem pode cadastrar e editar quem (Fase 27)

Os endpoints acima nasceram protegidos só por escopo: quem tivesse `users:write` fazia tudo. A Fase 27 acrescentou a pergunta que faltava — *quem é esta pessoa?* — e com ela as regras:

```
MANAGER   criando MANAGER / OPERATOR   -> 201
MANAGER   criando CUSTOMER             -> 403   (cliente nasce pela loja, não pelo painel)
máquina   criando CUSTOMER             -> 201
OPERATOR  criando qualquer um          -> 403   (nem tem users:write no escopo do papel)

MANAGER   editando outro MANAGER/OPERATOR  -> 200
MANAGER   editando um CUSTOMER             -> 403
qualquer  editando o PRÓPRIO registro      -> 200
```

O `DELETE` que anonimiza e o `/me` continuam como estavam. Detalhes e o fluxo completo em [RBAC e controle de acesso](./rbac-e-controle-de-acesso.md).

---

## 🔄 A senha temporária, resolvida (Fase 30)

O `System.out.println(tempPassword)` descrito acima morreu — e não por ter virado log. Ele **deixou de fazer sentido**: o cadastro passou a gerar uma senha aleatória que ninguém precisa conhecer, e a senha de verdade é definida pelo próprio usuário, por um link enviado no e-mail de ativação.

```java
user.setPassword(passwordManager.encrypt(passwordManager.generate()));  // inútil de propósito
String plainToken = user.generateVerificationToken(activationTtl, tokenHasher);
emailSender.sendActivationEmail(user, plainToken);
```

E o `isDisabled()` fecha a porta enquanto isso não acontece: quem não clicou no link tem conta e não entra. Ver [Verificação de e-mail e troca de senha](./verificacao-de-email-e-troca-de-senha.md).

---

## 🔄 O `/me` completou o ciclo: atualizar e excluir o próprio perfil

O `/me` desta fase só lia. O ciclo fechou com `PUT` e `DELETE` — e três decisões que valem registro:

**O self-update tem DTO próprio.** `MyUserUpdateInput` carrega só `name`. O administrativo (`AuthUserUpdateInput`) carrega `name`, `type`, `enabled`. Não é o mesmo input com campos ignorados: é a afirmação, **em tipo**, de que ninguém edita o próprio papel nem se auto-habilita. No serviço, uma sobrecarga `update(UUID, MyUserUpdateInput)` convive com a administrativa.

**Excluir o próprio perfil é privilégio de CUSTOMER.** A anotação compõe as duas condições que este documento apresentou separadas:

```java
@PreAuthorize("@securityCheck.canAccessOwnProfile() and hasRole('CUSTOMER')")
public @interface CanDeleteOwnProfile {}
```

Um MANAGER que se excluísse pelo `/me` deixaria o back-office sem administrador por um clique — conta interna é excluída por outro MANAGER, pela rota administrativa.

**E a camada de aplicação pergunta de novo** — porque o mesmo `anonymize` serve o `DELETE /users/{userId}` administrativo, onde um MANAGER poderia tentar excluir a si mesmo por outra porta:

```java
if (securityCheck.isMachineAuthenticated()) {
    return;                                   // ← m2m é sempre administrativo; sem "próprio perfil"
}
if (securityCheck.getAuthenticatedUserId().equals(user.getId())
        && user.getType() != AuthUserType.CUSTOMER) {
    throw new AccessDeniedException("Only CUSTOMER users can delete their own profile");
}
```

A primeira versão desse guard não tinha o `return` de máquina — e `getAuthenticatedUserId()` **lança** para `client_credentials`. O client m2m que tinha acabado de ganhar `users:write` levaria 403 num fluxo legítimo. O guard de aplicação que consulta identidade precisa perguntar *"é máquina?"* antes de perguntar *"quem é?"*.

O padrão `/me` inteiro, atravessando os quatro serviços, está em [Recursos `/me` e IDOR](./recursos-me-e-idor.md).

---

## Filtros e paginação

Mesmo desenho dos outros serviços (ver [`paginacao.md`](../02-persistencia/paginacao.md)): `AuthUserFilter extends SortablePageFilter`, Criteria API com predicados condicionais, `PageModel` de saída.

```
?email=ALGASHOP.COM          → 4 resultados   (LIKE parcial, case-insensitive)
?type=MANAGER                → 1
?type=OPERATOR&size=1&page=0 → total=3, paginas=3, pagina=0
?type=OPERATOR&size=1&page=1 → total=3, paginas=3, pagina=1  (registro diferente)
?email=naoexiste@x.com       → total=0, paginas=0
```

A cobertura automatizada disso é o `AuthUserQueryServiceIT`, contra Postgres real — o porquê está em [`testes-integracao-query-services.md`](../03-testes-integracao/testes-integracao-query-services.md).

---

## O achado: três bugs encadeados

`./gradlew check` chegou a esta fase com **27 falhas no `ordering`** e **2 no `billing`**. Uma causa só na origem, três sintomas.

### (a) O NPE que derrubava toda escrita

```
java.lang.NullPointerException: Cannot invoke "java.util.List.contains(Object)"
  because the return value of "org.springframework.security.oauth2.jwt.Jwt.getAudience()" is null
```

`aud` é um claim **opcional** no JWT. Quando não vem, `getAudience()` devolve `null` — e `isMachineAuthenticated()` chamava `.contains()` direto nele.

O detalhe que multiplica o estrago: quem chama esse método é o `AuditorAware`, **antes de cada persistência**. Não é um endpoint que quebra; é todo `POST`, todo `PUT`, todo `DELETE` de todos os serviços, virando 500 onde o teste esperava 201 ou 204.

### (b) O `catch` que não pegava nada

```java
try {
    return UUID.fromString(jwt.getSubject());
} catch (IllegalAccessError e) {          // ← IllegalAccessError, não IllegalArgumentException
    throw new AuthorizationDeniedException("Invalid user ID in JWT subject");
}
```

`UUID.fromString` lança **`IllegalArgumentException`**. `IllegalAccessError` é um `Error` de linkagem da JVM — acontece quando um `.class` compilado contra uma versão acessa membro que virou privado em outra. Ali dentro, **nunca**.

O bloco compilava, parecia proteção e era código morto. Um `sub` malformado viraria 500 em vez do 403 pretendido. Não tinha aparecido ainda porque (a) falhava antes — e apareceria no instante em que (a) fosse corrigido, porque o mock de teste do `ordering` usava `sub = "test-user"`, que não é UUID.

> Os dois bugs estavam nas **quatro** cópias do arquivo. Foi a duplicação cobrando: quatro edições idênticas para um conserto só.

### (c) A asserção que envelheceu

`InvoiceManagementApplicationServiceIT` afirmava `getCreatedByUserId()).isNotNull()`. Com `UUID.randomUUID()` isso passava **sempre**, em qualquer contexto — a asserção nunca teve chance de falhar e por isso nunca provou nada. Com o comportamento novo, e os ITs rodando sem autenticação, o autor passou a ser nulo corretamente e a asserção caiu.

O conserto não foi "fazer o teste passar", foi trocar o que ele afirma:

```java
// sem usuário autenticado, o AuditorAware devolve Optional.empty() — de propósito
assertThat(invoice.getCreatedByUserId()).isNull();
```

E no `ordering`, onde as fatias `@DataJpaTest` também quebraram (o `SpringDataAuditingConfig` passou a depender de um `@Service` que a fatia não carrega), a correção foi um stub com **UUID conhecido** — o que permitiu **fortalecer** a asserção em vez de afrouxá-la:

```java
assertThat(entity.getCreatedByUserId()).isEqualTo(TestSecurityCheckConfig.TEST_USER_ID);
```

Com `isNotNull()` o teste passaria igual se a auditoria voltasse a gravar `UUID.randomUUID()`. Com `isEqualTo`, não.

### E um quarto, no authorization server

Os 8 testes do `AuthUserQueryServiceIT` não subiam o contexto:

```
No qualifying bean of type 'RegisteredClientRepository' available
```

`TestSecurityConfig` existia, estava versionado, declarava exatamente esse bean — e **ninguém o importava**. `@TestConfiguration` não entra por component scan; só por `@Import` explícito. O arquivo compilava e não fazia nada. Mesma família do `@Configuration` perdido no rename da Fase 24: uma anotação ausente não é erro de compilação.

Corrigido o `@Import`, apareceu o bean seguinte da cadeia — `JwtDecoder`, que vinha da mesma auto-configuração desligada. Vale entender por que ela está desligada: o `OAuth2AuthorizationServerAutoConfiguration` só age quando existem propriedades `spring.security.oauth2.authorizationserver.client.*`, e elas moram só no `application-development-env.yaml`, que o grupo de perfis `test` não carrega. Sem ela somem, em cascata, **três** beans que o `AuthorizationServerSecurityConfig` exige.

---

## Armadilhas e pendências

**A senha temporária vaza e não chega a ninguém.** *(✅ resolvido na Fase 30 — ver adiante.)* `create()` gera 12 caracteres aleatórios, imprime no stdout e guarda só o hash:

```java
String tempPassword = RandomStringUtils.secure().nextAlphabetic(12);
System.out.println(tempPassword);          // ← credencial em texto puro no log
String passwordHash = passwordEncoder.encode(tempPassword);
```

Verificado: dois usuários criados, duas senhas em claro no stdout do servidor, `{bcrypt}$2a$…` no banco, e **nada** no corpo da resposta. O usuário criado pela API **não consegue logar** — o cadastro está incompleto, não só inseguro. Falta o canal de entrega (e-mail com link de definição de senha) e sobra o `println`. Mantido como está nesta fase, por decisão; registrado aqui com o cenário.

**O nome do bean é contrato.** `@Service("securityCheck")` e `@PreAuthorize("@securityCheck…")` — ver acima.

**A porta está duplicada nos quatro serviços**, byte a byte. Decisão consciente; custo demonstrado nesta fase.

**`isMachineAuthenticated()` é heurística.** Um claim explícito no token seria mais robusto.

**Não há `AuthorizationMatrixTest` no authorization server.** *(✅ resolvida — a matriz existe: importa a filter chain real com os beans OAuth2 mockados e a implementação real do bean `securityCheck`. Ver [Recursos `/me` e IDOR](./recursos-me-e-idor.md).)* Os outros três serviços têm; aqui o `@WebMvcTest` arrastaria a filter chain do protocolo OAuth2 inteira. Detalhado em [`testes-integracao-query-services.md`](../03-testes-integracao/testes-integracao-query-services.md#pendência-registrada).

**A auditoria de máquina é anônima por design** — se um dia for preciso rastrear qual sistema escreveu, é coluna nova.

---

## Resumo mental

> **Identidade que atravessa a fronteira é o que torna auditoria possível.** Sem `sub` no token, "quem criou" só podia ser placeholder.
>
> **`/me` elimina uma classe de bug em vez de proteger contra ela** — o id que o cliente não escolhe não pode ser o id de outra pessoa.
>
> **`Optional.empty()` para máquina é dado; `UUID.randomUUID()` era ruído com cara de dado.**
>
> **Anonimizar preserva a integridade que apagar destruiria** — o autor registrado em outros serviços continua resolvendo.
>
> **`catch` de exceção que não acontece é comentário que compila** — e o compilador aceita porque `IllegalAccessError` é alcançável em teoria.
>
> **Anotação ausente não é erro de compilação** — `@Configuration`, `@Import`, nome de bean em SpEL: tudo isso falha em runtime, longe da causa.
