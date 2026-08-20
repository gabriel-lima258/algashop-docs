# Testando segurança: identidade em teste, e o que o mock esconde

O caderno tem quatro documentos sobre **o que** a segurança do AlgaShop faz — [escopos](../05-seguranca/resource-server-e-escopos.md), [gestão de usuários](../05-seguranca/gestao-de-usuarios-e-auditoria.md), [RBAC](../05-seguranca/rbac-e-controle-de-acesso.md), [telas](../05-seguranca/telas-e-formularios-de-login.md) — e nenhum sobre **como testá-la**. Este preenche a lacuna.

O fio condutor é uma pergunta só: **o que este teste realmente prova?** Porque em teste de segurança é fácil escrever algo verde que não prova nada.

---

## Três maneiras de pôr identidade num teste

As regras das fases 27 e 28 dependem de *quem está chamando*. Para testá-las, o `SecurityContext` precisa ter alguém dentro. Há três caminhos, e eles não são equivalentes:

### 1. Mockar a porta — a tautologia

```java
@MockitoBean SecurityCheckApplicationService securityCheck;
Mockito.when(securityCheck.isCustomer()).thenReturn(true);
```

O teste passa a afirmar: *"quando eu digo que é um CUSTOMER, o serviço trata como CUSTOMER"*. Nada sobre o token, o converter, o prefixo `ROLE_` ou a heurística de máquina. Se o `JwtGrantedAuthoritiesDelegatingConverter` parar de somar o papel amanhã, este teste continua verde.

> **Mockar a porta testa quem a chama, não quem a implementa.** Serve para o `BuyNowApplicationServiceIT` — que quer testar checkout, não segurança. Não serve para testar a segurança em si.

É o que o `AbstractPersistenceIT` faz, e ali está certo: aquelas fatias verificam **auditoria**, e o adapter real nem estaria no contexto de um `@DataJpaTest`.

### 2. Montar a autenticação — imperativo

```java
TestAuthentications.authenticateAsCustomer(customerId);
```

O `Jwt` é montado com os mesmos claims que o `JwtTokenCustomizer` do authorization server escreve e passa pelo **mesmo converter** de produção. Prova de verdade — mas espalha setup em `@BeforeEach`, e a identidade do teste não aparece na assinatura dele.

### 3. `@WithSecurityContext` — declarativo

```java
@WithMockJwt(role = "MANAGER")
class OrderQueryServiceIT extends AbstractIntegrationTest { ... }
```

A mesma prova, dita na assinatura. É o mecanismo que o Spring Security oferece exatamente para isso.

---

## `@WithMockJwt` por dentro

Duas peças:

```java
@Retention(RetentionPolicy.RUNTIME)
@WithSecurityContext(factory = WithMockJwtSecurityContextFactory.class)
public @interface WithMockJwt {
    String subject()   default "6e148bd5-47f6-4022-b9da-07cfaa294f7a";
    String[] scopes()  default { "orders:read", "orders:write", ... };
    String role()      default "CUSTOMER";
    String[] audiences() default { "ecommerce-web-app" };
}
```

```java
@Component
public class WithMockJwtSecurityContextFactory implements WithSecurityContextFactory<WithMockJwt> {

    private final JwtAuthenticationConverter jwtAuthenticationConverter;   // ← o do CONTEXTO

    public SecurityContext createSecurityContext(WithMockJwt annotation) {
        Jwt jwt = MockJwtFactory.buildJwt(..., annotation.role(), annotation.audiences());
        AbstractAuthenticationToken token = jwtAuthenticationConverter.convert(jwt);
        SecurityContext context = SecurityContextHolder.createEmptyContext();
        context.setAuthentication(token);
        return context;
    }
}
```

**O detalhe que decide o valor de tudo isto** está na terceira linha: o converter é **injetado**, não instanciado. É o mesmo bean que o resource server usa em produção — aquele que a Fase 27 configurou para somar `ROLE_*` ao lado dos `SCOPE_*`. Trocá-lo por um `new JwtAuthenticationConverter()` faria o teste passar por um caminho que produção não percorre, e o mecanismo inteiro deixaria de estar coberto.

### O truque do papel vazio

```java
@WithMockJwt(role = "", audiences = "machine-client-id", subject = "machine-client-id")
```

É assim que se simula `client_credentials`, e ele funciona por duas coincidências deliberadas:

- o converter faz `StringUtils.isNotBlank(role)` antes de somar a authority — papel vazio não vira `ROLE_*`;
- `isMachineAuthenticated()` deduz máquina de `aud` **conter** `sub` — daí subject e audience iguais.

Elegante, e frágil exatamente onde a heurística já era frágil. Se um dia o token ganhar um claim explícito de tipo (a pendência registrada na Fase 25), esta anotação muda junto.

---

## Declarativo × imperativo: o critério

Os dois mecanismos convivem, e a escolha tem uma regra clara:

| A identidade… | Use | Exemplo |
|---|---|---|
| existe **antes** do teste rodar | `@WithMockJwt` | `OrderQueryServiceIT` é MANAGER; `SecurityCheckApplicationServiceIT` varia por método |
| é **criada pelo** teste | `TestAuthentications` | `CheckoutApplicationServiceIT` persiste um cliente novo e precisa autenticar como ele |

```java
private CustomerId persistCustomer() {
    CustomerId customerId = new CustomerId();
    customers.add(CustomerTestDataBuilder.existingCustomer().id(customerId).build());
    TestAuthentications.authenticateAsCustomer(customerId.value());   // o id só existe agora
    return customerId;
}
```

Uma anotação não alcança esse caso: ela é resolvida antes do método rodar, e o id ainda não existe. **O imperativo não é o jeito antigo — é o jeito do caso dinâmico.**

E a anotação **herda e sobrescreve**: `@WithMockJwt` no `AbstractIntegrationTest` vale para a hierarquia toda; declará-la de novo na subclasse substitui. É o que permite ao caso comum ficar num lugar só.

---

## O que o mock **não** testa

Esta é a seção mais importante do documento, porque a suíte verde sugere mais cobertura do que existe.

```java
Mockito.when(jwtDecoder.decode(DEFAULT_TOKEN_VALUE)).thenReturn(buildDefaultJwt());
```

O `MockJwtFactory` substitui o `JwtDecoder` **inteiro**. O objeto `Jwt` já sai pronto, então nada disto é verificado:

| | Coberto? |
|---|---|
| assinatura do token | ❌ nunca é conferida |
| `iss` bate com o issuer configurado | ❌ |
| `exp` no passado é rejeitado | ❌ — o caso "expirado" é um `thenThrow(JwtException)`, que exercita o **tratamento** da falha, não a detecção |
| `aud` é validada | ❌ (e não é validada em produção também — pendência da Fase 21) |
| escopo vira `SCOPE_*` | ✅ |
| papel vira `ROLE_*` | ✅ |
| `@PreAuthorize` decide 401/403 | ✅ |
| regra de dono do recurso | ✅ |

> **A suíte cobre AUTORIZAÇÃO, não AUTENTICAÇÃO.** Validação real de token só acontece contra o authorization server rodando — e é por isso que as fases de segurança sempre terminam com uma prova por `curl`, e não só com a suíte verde.

Saber exatamente onde a cobertura acaba vale mais do que ampliá-la: um teste que valida assinatura precisaria de chave, e provaria coisa que a biblioteca já garante.

---

## Testabilidade é propriedade do desenho

Os ITs de apresentação não subiam. O motivo não era o teste — era o desenho:

```java
// antes: o interceptor nascia DENTRO do método que criava o client
public ProductCatalogApiClient productCatalogApiClient(..., OAuth2AuthorizedClientManager manager) {
    OAuth2ClientHttpRequestInterceptor interceptor = new OAuth2ClientHttpRequestInterceptor(manager);
    ...
}
```

Não havia bean para substituir. Todo IT de apresentação tentava buscar token de verdade no authorization server — que não está de pé durante os testes. A correção foi extrair:

```java
@Bean("productCatalogAPIClientInterceptor")
public OAuth2ClientHttpRequestInterceptor productCatalogAPIClientInterceptor(...) { ... }
```

```java
// AbstractPresentationIT
@MockitoBean("productCatalogAPIClientInterceptor")
protected OAuth2ClientHttpRequestInterceptor productCatalogAPIClientInterceptor;
```

> **Dependência criada dentro de um método não se substitui em teste.** Ela não é um bean, não tem nome, e nada externo a alcança. O que destravou os testes foi uma mudança de *desenho* — e é o argumento mais concreto a favor de injetar em vez de instanciar.

Repare que a extração não deixou o código de produção mais complicado: o mesmo objeto, no mesmo lugar do fluxo, apenas com um nome.

---

## As três camadas de teste de segurança no `ordering`

| Camada | Ferramenta | Alcança | Não alcança |
|---|---|---|---|
| **Matriz** (`AuthorizationMatrixTest`) | `@WebMvcTest` + `jwt()` post-processor | rota × escopo, 401 × 403, para toda a superfície | regra de negócio, dono do recurso |
| **Aplicação** (`*ApplicationServiceIT`) | `@SpringBootTest` + `@WithMockJwt` | regra de dono, papel, o adapter real | a camada HTTP |
| **Apresentação** (`*ControllerIT`) | RestAssured + token de mentira | o caminho inteiro, do header ao banco | validação do token |

Nenhuma cobre tudo, e é essa a intenção: a matriz é larga e rasa, a de aplicação é estreita e funda, e a de apresentação prova que as duas se encontram.

---

## O achado: a asserção que afrouxou contra o próprio comentário

Ao trocar o `TestSecurityCheckConfig` por um `@MockitoBean`, o stub passou a responder:

```java
Mockito.when(securityCheck.getAuthenticatedUserId()).thenReturn(UUID.randomUUID());
```

E a asserção de auditoria, que a Fase 25 tinha **fortalecido**, voltou ao que era:

```java
- assertThat(entity.getCreatedByUserId()).isEqualTo(TestSecurityCheckConfig.TEST_USER_ID);
+ assertThat(entity.getCreatedByUserId()).isNotNull();
```

O detalhe que faz disto um achado e não um descuido qualquer: **o comentário logo acima continuou lá**, dizendo *"com `isNotNull()` ele passaria igual se a auditoria voltasse a gravar `UUID.randomUUID()`"*. O teste passou a fazer exatamente aquilo contra o que ele próprio avisa — e o mock passou a devolver, literalmente, um `UUID.randomUUID()`.

A correção custou duas linhas: um `TEST_USER_ID` fixo no mock, e o `isEqualTo` de volta.

> **Refatoração de teste carrega risco que refatoração de produção não carrega**: quando um teste enfraquece, nada fica vermelho. A rede que avisaria é o próprio teste.

O sinal de alerta útil aqui foi o comentário sobrevivendo à mudança que ele descrevia. **Comentário que contradiz o código ao lado costuma marcar o lugar exato onde algo se perdeu.**

---

## A mutação: os testes têm dentes (e um limite)

Apagando o claim `role` do `MockJwtFactory` — a linha de que todo o mecanismo depende:

```
givenAuthenticatedCustomerShouldAllowOrderForHimself()      FAILED
authenticatedCustomerShouldCarryTheRoleAuthority()          FAILED

givenAuthenticatedManagerShouldNotBeCustomer()              PASSED
givenAuthenticatedManagerShouldDenyOrder()                  PASSED
givenAuthenticatedMachineShouldBeDetectedAsMachine()        PASSED
```

Dois vermelhos, e os certos. Mas repare em **quais passaram**: todos os de asserção negativa. Sem papel nenhum, `isCustomer()` é falso — que é exatamente o que eles afirmam. Eles passam **pelo motivo errado**.

> **Teste de asserção negativa não detecta mecanismo quebrado.** Ele fica verde tanto quando a regra funciona quanto quando ela nunca é alcançada. Só o caso positivo distingue os dois.

É a mesma lição da Fase 28, onde o teste que fazia login continuou verde com o formulário quebrado. Uma suíte de segurança que só afirma "não pode" não descobre que ninguém pode nada.

---

## Armadilhas

- **Mockar a porta em teste de segurança** produz tautologia — mocke-a em teste de *outra coisa*.
- **Instanciar o converter no factory** em vez de injetá-lo tira do teste justamente o que ele deveria cobrir.
- **`role = ""`** simula máquina por dois acidentes felizes; muda junto com a heurística.
- **Suíte verde ≠ token validado** — assinatura, `iss` e `exp` nunca são conferidos.
- **Só asserções negativas** dão falso conforto.
- **Build incremental esconde classe deletada** — o `compileTestJava` passou com dois imports de uma classe que não existia mais; só `clean` acusou.
- **`@ExtendWith(MockitoExtension.class)` junto com `@MockitoBean`** é redundante: o `@MockitoBean` é do Spring Test e não depende da extensão.

## Pendências registradas

- [ ] **Nenhum teste valida token de verdade** — nem aqui, nem em serviço nenhum. A prova é sempre manual, por `curl`.
- [ ] **A heurística `aud`/`sub`** continua sem claim explícito, e agora tem testes que dependem dela.
- [ ] **Os outros três serviços não têm equivalente do `@WithMockJwt`** — catálogo e billing testam autorização só pela matriz.
- [ ] **`TestAuthentications` duplica o que a factory faz**, com um converter instanciado à mão em vez do bean.

## Checklist de revisão

- [ ] Este teste prova algo sobre a segurança, ou só sobre o que eu mandei o mock responder?
- [ ] O converter usado é o **do contexto**?
- [ ] Há pelo menos uma asserção **positiva** exercitando o mecanismo?
- [ ] A identidade existe antes do teste (declarativo) ou é criada por ele (imperativo)?
- [ ] O que este mock esconde está escrito em algum lugar?

## Referências

- [Spring Security — Testing Method Security](https://docs.spring.io/spring-security/reference/servlet/test/method.html)
- [`@WithSecurityContext`](https://docs.spring.io/spring-security/reference/servlet/test/method.html#test-method-withsecuritycontext)
- [Testes de integração de query services](./testes-integracao-query-services.md) · [Stubs e contract tests](./stubs-contract-tests.md) · [RBAC e controle de acesso](../05-seguranca/rbac-e-controle-de-acesso.md)
