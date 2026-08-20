# Testes de integração de query services

Este documento explica como o AlgaShop testa os **query services** — a metade de consulta do CQRS — contra banco de dados real, usando Testcontainers. Os dois exemplos vivos são o `CustomerQueryServiceIT` (algashop-ordering, Postgres) e o `AuthUserQueryServiceIT` (authorization-server, Postgres).

---

## Por que banco de verdade, e não mock?

Um query service como o `AuthUserQueryServiceImpl` é, essencialmente, **construção de SQL**: Criteria API, predicados condicionais, `LIKE lower(...)`, projeção com `builder.construct`, `setFirstResult`/`setMaxResults`. Ver [`paginacao.md`](../02-persistencia/paginacao.md).

Mockar o `EntityManager` aqui testaria apenas que o código chama o `EntityManager` — não que a query está **certa**. As perguntas que importam só o banco responde:

| Pergunta | Quem responde |
|---|---|
| O `LIKE` com `lower()` é realmente case-insensitive? | o banco |
| `size=1, page=1` devolve o segundo registro, não o primeiro de novo? | o banco |
| A ordem dos argumentos do `builder.construct` casa com o construtor do DTO? | o runtime do Hibernate |
| O filtro por enum compara pelo valor persistido (`@Enumerated(STRING)`)? | o banco |
| `type` chama o service que chama o repositório? | isso um unitário resolve — e não precisa deste teste |

> **Lição do projeto:** o erro de ordem no `builder.construct` compila, o contexto sobe, e só estoura quando a query executa. É a classe de bug que unitário nenhum pega — e o motivo de esta suíte existir.

O complemento barato é o teste **unitário de construção de query** (sem banco): o `ProductQueryServiceImplTest` do product-catalog usa `@Mock` + `ArgumentCaptor<Query>` para inspecionar o critério gerado e cobrir combinações de predicados. Os dois se somam; não competem.

---

## A anatomia do teste

```java
@SpringBootTest
@Transactional
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Import(TestContainerPostgresSQLConfig.class)
class AuthUserQueryServiceIT {

    @Autowired
    private AuthUserQueryService authUserQueryService;   // a interface, nunca a impl

    @Autowired
    private AuthUserRepository authUserRepository;       // para plantar os dados

    @BeforeEach
    void setUp() {
        johnManager = authUserRepository.save(AuthUserTestDataBuilder.aManagerUser());
        aliceOperator = authUserRepository.save(AuthUserTestDataBuilder.anOperatorUser());
        // ...
    }
}
```

Cada anotação tem um trabalho:

| Anotação | O que faz | O que acontece sem ela |
|---|---|---|
| `@SpringBootTest` | sobe o contexto completo da aplicação | — |
| `@Transactional` | abre transação por teste e faz **rollback** no final | dados de um teste vazam para o próximo |
| `@AutoConfigureTestDatabase(replace = NONE)` | impede o Boot de trocar o datasource por um H2 embutido | a suíte testaria contra H2, não Postgres |
| `@Import(TestContainerPostgresSQLConfig.class)` | sobe o Postgres efêmero e injeta a conexão | o teste dependeria do compose de pé |

### `webEnvironment`: a diferença entre o ordering e o authorization-server

O `AbstractIntegrationTest` do ordering usa `@SpringBootTest(webEnvironment = NONE)` — o teste não faz HTTP, então não sobe servidor. É a escolha certa lá.

No authorization-server **isso quebra o contexto**. A autoconfiguração do Spring Authorization Server — que materializa o `RegisteredClientRepository` a partir dos clientes declarados no YAML — só roda em aplicação **servlet**. Com `NONE`, ela some; mas as `@Configuration` do serviço que dependem desse bean (`OAuth2PersistenceConfig`, handlers de logout OIDC) continuam carregando, e o contexto morre com `NoSuchBeanDefinitionException`.

> **Lição:** `webEnvironment = NONE` não é um default universal de IT. Ele assume que nada no contexto **precisa** ser web — e um authorization server precisa. O sintoma (bean do framework sumindo) aparece longe da causa (o tipo de contexto).

---

## Isolamento de dados: o seed é o inimigo silencioso

Os testes de listagem contam registros: `totalElements=2`, `totalPages=2`. Qualquer linha que o teste **não criou** quebra a conta. E há uma fonte de linhas fácil de esquecer: o **seed do Flyway**.

O grupo de perfis de teste inclui `development-env` de propósito (para validar o YAML dos clientes OAuth2 no `contextLoads`). Só que o `development-env` também traz:

```yaml
spring:
  flyway:
    locations: classpath:db/migration,classpath:db/testdata   # testdata = seed!
```

O `afterMigrate.sql` de `db/testdata` insere 3 usuários. E o Flyway roda **fora** da transação do teste — o rollback do `@Transactional` não desfaz nada disso.

```
        Flyway (na subida do contexto)          @Transactional (por teste)
  ┌────────────────────────────────────┐   ┌────────────────────────────────┐
  │ migrations + afterMigrate (seed)   │   │ BEGIN → inserts do teste →     │
  │ COMMIT — fica no banco             │   │ asserts → ROLLBACK             │
  └────────────────────────────────────┘   └────────────────────────────────┘
         sobrevive à suíte inteira               some ao fim de cada teste
```

Cada serviço resolveu de um jeito, e a diferença é instrutiva:

| Serviço | Estratégia | Como |
|---|---|---|
| algashop-ordering | **limpar depois** | perfil test troca o location para `db/clean`, cujo `afterMigrate.sql` faz `TRUNCATE CASCADE` |
| authorization-server | **não sujar** | perfil test sobrescreve `locations: classpath:db/migration` — sem `db/testdata`, o banco nasce vazio |

A segunda é mais simples (zero arquivo novo), mas só é possível porque nada no contexto depende do seed para subir. No ordering, o location precisava continuar existindo — daí o `db/clean`.

A terceira opção — escrever os testes "convivendo" com o seed — foi descartada nos dois serviços: cada INSERT novo no seed quebraria asserções de contagem em testes que nem olham para ele.

---

## Test data builders

Os dados nascem no `@BeforeEach`, por builders de teste que passam pela **fábrica do domínio** (não por SQL nem por setters soltos):

```java
public class AuthUserTestDataBuilder {
    public static AuthUser aManagerUser() {
        return AuthUser.brandNew("john.manager@algashop.com", "John Manager",
                AuthUserType.MANAGER, PASSWORD_HASH);
    }
}
```

Passar pela fábrica (`brandNew`) garante que o dado de teste respeita as invariantes do agregado — id UUIDv7 gerado, `enabled=true`, setters validando blank. Um INSERT manual poderia criar um estado que o domínio jamais produziria, e o teste validaria um cenário impossível.

Convenções (as mesmas do `CustomerTestDataBuilder`, `CategoryDetailOutputTestDataBuilder` etc.):
- e-mails com **domínios distinguíveis** (`@algashop.com` vs `@gmail.com`) para exercitar o `LIKE` parcial;
- nomes que tornam a ordenação verificável (`Alice` < `Carol` em `NAME ASC`);
- asserções AssertJ com `extracting(...)` + `containsExactlyInAnyOrder` (filtro) ou `containsExactly` (quando a ordem é o que se testa).

---

## Os cenários mínimos de um query service

O `AuthUserQueryServiceIT` cobre o conjunto que todo query service paginado deveria ter:

| Cenário | O que prova |
|---|---|
| `shouldFindById` | projeção campo a campo bate com o registro persistido |
| `shouldThrowNotFoundWhenUserDoesNotExist` | id inexistente vira a exceção do domínio de consulta (que o `ApiExceptionHandler` traduz em 404) |
| filtro por e-mail (parcial + case-insensitive) | o predicado `LIKE lower()` funciona no banco real |
| filtro por tipo com `size=1`, páginas 0 e 1 | offset/limit corretos, `totalElements`/`totalPages`/`number` consistentes |
| filtro sem correspondência | página vazia com `totalElements=0` (e o early-return do count zero) |

---

## A suíte `*IT` no Gradle

Os ITs vivem no mesmo `src/test`, separados por **sufixo de nome**, não por source set:

```groovy
test {
    filter { excludeTestsMatching("*IT") }   // ./gradlew test roda sem Docker
}

tasks.register('integrationTest', Test) {
    shouldRunAfter test                      // ordem, não dependência
    filter {
        includeTestsMatching "*IT"
        excludeTestsMatching "*Test"
    }
}

tasks.named('check') {
    dependsOn(test, contractTest, integrationTest)
}
```

A separação completa (motivos, `mockitoAgent`, `shouldRunAfter` vs `dependsOn`) está documentada em [`stubs-contract-tests.md`](./stubs-contract-tests.md#separando-unidade-contrato-e-integração-no-gradle).

```
./gradlew test              # sem Docker
./gradlew integrationTest   # exige Docker (Testcontainers)
./gradlew check             # amarra as três suítes
```

---

## Pendência registrada

O authorization-server ainda **não tem** o `AuthorizationMatrixTest` que os outros três serviços têm (sem token → 401, escopo errado → 403, escopo certo → passa). O bloqueio é real: importar o `AuthorizationServerSecurityConfig` num `@WebMvcTest` arrasta a filter chain do protocolo OAuth2 inteira, que exige meia dúzia de beans do Spring Authorization Server. A matriz para `/api/**` vale a pena, mas pede uma `@TestConfiguration` que suba só a chain `@Order(2)` — investigação própria, fora do escopo da leva que criou esta suíte.

---

## E quando o teste envolve segurança

Um query service protegido por papel (o `OrderQueryService` do `ordering`, que filtra por dono) precisa de identidade no contexto — e aí entram decisões que este documento não cobre: mockar a porta, montar a autenticação ou declarar com `@WithMockJwt`. Ver [Testando segurança](./testando-seguranca.md).

---

## Resumo mental

> **Query service se testa contra banco real** — o que está em jogo é o SQL gerado, não a orquestração.
>
> **Seed do Flyway não respeita rollback** — ou o perfil de teste limpa (`db/clean` + TRUNCATE), ou não semeia (override de `locations`).
>
> **`@Transactional` no teste = isolamento grátis** — cada teste planta seus dados e o rollback varre.
>
> **Builder que passa pela fábrica do domínio** — dado de teste com as mesmas invariantes do dado de produção.
>
> **`webEnvironment = NONE` é otimização, não default** — se algo no contexto precisa ser web (um authorization server, por exemplo), ela derruba a subida.
