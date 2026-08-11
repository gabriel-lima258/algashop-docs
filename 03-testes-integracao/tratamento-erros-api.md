# Tratamento de erros na API REST

> Como o `product-catalog` traduz exceções de domínio em respostas HTTP usando `ProblemDetail` (RFC 9457).
> O mesmo desenho vale para o `ordering` e o `billing`.

---

## O problema

Uma API que devolve isso quando algo dá errado é inútil para quem consome:

```
HTTP/1.1 500 Internal Server Error
Content-Type: text/html

<html><body><h1>Whitelabel Error Page</h1>...</body></html>
```

O cliente não sabe se a culpa é dele ou do servidor, não sabe se vale a pena tentar de novo, e não consegue mostrar mensagem útil ao usuário final. Precisamos de três coisas:

1. **Status HTTP correto** — 404 é diferente de 422, que é diferente de 500.
2. **Corpo previsível e parseável** — sempre o mesmo formato.
3. **O domínio não pode saber o que é HTTP** — a regra de negócio não deveria importar `HttpStatus`.

---

## A ideia central: exceção de domínio não conhece HTTP

Este é o ponto que sustenta todo o resto.

```java
// domain/product/ProductNotFoundException.java
public class ProductNotFoundException extends DomainEntityNotFoundException {
    public ProductNotFoundException(UUID productId) {
        super(String.format("Product with id %s was not found", productId));
    }
}
```

Nenhuma menção a `404`, `HttpStatus` ou `ResponseStatusException`. A camada de domínio diz **o que aconteceu no negócio** ("produto não existe"); quem decide **como isso vira uma resposta HTTP** é a borda da aplicação.

Por que isso importa na prática: o mesmo `ProductManagementApplicationService` pode ser chamado por um controller REST, por um listener de mensageria ou por um job agendado. Só um desses três tem status HTTP. Se a exceção carregasse o `404`, o domínio estaria acoplado a um detalhe de entrega.

---

## A hierarquia de exceções

```
RuntimeException
├── DomainException                          → 422 Unprocessable Content
│      (regra de negócio violada)
│
├── DomainEntityNotFoundException            → 404 Not Found
│   ├── ProductNotFoundException
│   └── CategoryNotFoundException
│
├── ResourceNotFoundException                → 404 Not Found
│      (camada de aplicação)
│
└── UnprocessableContentException            → 422 Unprocessable Content

StorageProviderException (infrastructure/)   → 422 Unprocessable Content
       (camada de apresentação)
```

Duas famílias, três camadas:

| Exceção | Camada | Significado |
|---|---|---|
| `DomainException` | `domain/` | uma **regra de negócio** foi violada |
| `DomainEntityNotFoundException` | `domain/` | uma entidade referenciada **não existe** |
| `ResourceNotFoundException` | `application/` | o recurso pedido pelo caso de uso não existe |
| `UnprocessableContentException` | `presentation/` | o corpo da requisição é sintaticamente válido, mas semanticamente impossível |
| `StorageProviderException` | `infrastructure/` | o provedor de arquivos recusou a operação — arquivo ausente, chave já em uso |

> `StorageProviderException` é a única das quatro que nasce na **infraestrutura**, e por isso merece um segundo de atenção. Ela cai em 422 e não em 500 porque o que ela relata quase sempre é decisão de quem chamou — pedir para anexar um arquivo que não está no bucket é entrada inválida, não falha do servidor. O risco embutido é o oposto: uma indisponibilidade real do S3 também vira 422, culpando o cliente por um problema que não é dele. Verificado na Fase 19: anexar `nunca-subiu.jpg` devolve `422` com `ProblemDetail` correto.

As classes base (`DomainException` e `DomainEntityNotFoundException`) reexpõem os cinco construtores de `RuntimeException`. É verboso, mas garante que qualquer subclasse possa escolher entre mensagem, causa ou ambos.

**Por que `RuntimeException` e não `Exception` (checked)?** Se fosse checked, toda a cadeia de chamadas — repositório, application service, controller — precisaria declarar `throws` ou envolver em `try/catch`. Regra de negócio violada não é algo que quem chama vai "tratar e continuar"; é algo que aborta a operação. Unchecked é o encaixe certo.

---

## O handler central

`@RestControllerAdvice` intercepta exceções de todos os controllers. O corpo é sempre um `ProblemDetail`, a implementação do Spring para a **RFC 9457** (antiga RFC 7807, *Problem Details for HTTP APIs*).

```java
// presentation/ApiExceptionHandler.java
@AllArgsConstructor
@RestControllerAdvice
@Slf4j
public class ApiExceptionHandler extends ResponseEntityExceptionHandler {
```

### 404 — um handler para várias exceções

```java
@ExceptionHandler({DomainEntityNotFoundException.class, ResourceNotFoundException.class})
public ProblemDetail handleNotFoundException(Exception e) {
    ProblemDetail problemDetail = ProblemDetail.forStatus(HttpStatus.NOT_FOUND);
    problemDetail.setTitle("Not found");
    problemDetail.setDetail(e.getMessage());
    problemDetail.setType(URI.create("/errors/not-found"));
    return problemDetail;
}
```

`@ExceptionHandler` aceita um **array** de classes. Como as duas exceções não têm ancestral comum além de `RuntimeException`, o parâmetro do método precisa ser o tipo comum mais próximo — daí `Exception e`. Agrupar assim evita dois métodos idênticos.

Repare que `ProductNotFoundException` e `CategoryNotFoundException` **não** precisam ser listadas: o Spring resolve por hierarquia, e as duas herdam de `DomainEntityNotFoundException`.

### 422 — conteúdo semanticamente inválido

```java
@ExceptionHandler({DomainException.class, UnprocessableContentException.class,
                   StorageProviderException.class})
public ProblemDetail handleUnprocessableContentException(Exception e) {
    ProblemDetail problemDetail = ProblemDetail.forStatus(HttpStatus.UNPROCESSABLE_CONTENT);
    problemDetail.setTitle("Unprocessable content");
    problemDetail.setDetail(e.getMessage());
    problemDetail.setType(URI.create("/errors/unprocessable-content"));
    return problemDetail;
}
```

**400 vs. 422** — a distinção que mais gera dúvida:

| | 400 Bad Request | 422 Unprocessable Content |
|---|---|---|
| Quando | O servidor não consegue **entender** a requisição | Entendeu perfeitamente, mas não consegue **processar** |
| Exemplo | JSON malformado, campo obrigatório ausente, tipo errado | `salePrice` maior que `regularPrice`; `categoryId` que não existe |
| Quem gera aqui | Bean Validation (`@NotBlank`, `@NotNull`) | `DomainException`, `UnprocessableContentException` |

### 400 — erros de validação campo a campo

O handler sobrescreve `handleMethodArgumentNotValid` para enriquecer o `ProblemDetail` com um mapa de erros por campo:

```java
Map<String, String> fieldErrors = ex.getBindingResult().getAllErrors().stream().collect(
        Collectors.toMap(
                objectError -> ((FieldError) objectError).getField(),
                objectError -> messageSource.getMessage(objectError, LocaleContextHolder.getLocale())
        )
);

problemDetail.setProperty("fields", fieldErrors);
```

`setProperty` adiciona campos **extras** ao JSON, que é exatamente o que a RFC 9457 permite (*extension members*). O `MessageSource` + `LocaleContextHolder` fazem as mensagens saírem no idioma do `Accept-Language` da requisição, em vez de hardcoded em inglês.

> ⚠️ `Collectors.toMap` lança `IllegalStateException` em chave duplicada. Se um mesmo campo acumular duas violações (ex.: `@NotBlank` **e** `@Size`), este código quebra — e o erro sai como 500. A correção seria passar a *merge function* (`(a, b) -> a + "; " + b`).

### 500 — a rede de segurança

```java
@ExceptionHandler(Exception.class)
public ProblemDetail handleException(Exception e) {
    log.error(e.getMessage(), e);                              // stacktrace só no log
    ProblemDetail problemDetail = ProblemDetail.forStatus(HttpStatus.INTERNAL_SERVER_ERROR);
    problemDetail.setDetail("An unexpected error occurred");   // mensagem genérica na resposta
    // ...
}
```

Duas decisões de segurança aqui, e ambas são deliberadas:

1. **O stacktrace vai para o log, nunca para a resposta.** Stacktrace exposto entrega nomes de classes, versões de biblioteca e estrutura interna para quem estiver sondando a API.
2. **A mensagem é genérica.** `e.getMessage()` de uma exceção inesperada pode conter fragmento de SQL, caminho de arquivo ou dado sensível.

Compare com o 404 e o 422, que **usam** `e.getMessage()` — ali a mensagem foi escrita por nós, é sobre o negócio, e é segura de mostrar.

---

### 502 e 504 — a falha que não é sua

Estes dois existem para não deixar problema de terceiro virar 500. A distinção entre eles é a mais útil de todo o conjunto:

| | Significa | O que se sabe |
|---|---|---|
| **502** Bad Gateway | a dependência **respondeu**, e a resposta não serve | a operação não aconteceu |
| **504** Gateway Timeout | a dependência **não respondeu** a tempo | **não se sabe** se aconteceu |

O 504 é o mais desconfortável dos dois, e é por isso que ele importa. Num `POST` que cobra dinheiro, timeout **não cancela** a autorização do outro lado — significa "não sei se cobrou". Devolver 500 ali apagaria essa informação; devolver 504 a preserva para quem for reconciliar depois.

> **Por que isso não pode ser 500.** O 5xx genérico diz "eu falhei". O 502/504 diz "quem eu chamei falhou" — e essa diferença é o que separa um alerta acionável de uma madrugada perdida procurando bug onde não há.

Na Fase 16 o `BadGatewayException` ganhou duas subclasses, e a razão é de resiliência, não de HTTP:

```java
public class BadGatewayException extends RuntimeException {
    public static class ServerErrorException extends BadGatewayException { }   // 5xx
    public static class ClientErrorException extends BadGatewayException { }   // 4xx
}
```

A `RetryPolicy` decide o que retentar por **assignability**, listando apenas `ServerErrorException`. Sem as subclasses não haveria como dizer "repita 5xx, não repita 4xx" — os dois seriam o mesmo tipo, e repetir um 401 daria 401 quatro vezes.

As três formas continuam virando 502 para o cliente: a distinção é interna, e só o retry a enxerga. Ver [`resiliencia.md`](../01-arquitetura-design/resiliencia.md).

---

## O caso interessante: 404 ou 422?

Este trecho do `ProductController` é o mais didático de todo o desenho:

```java
// presentation/ProductController.java
@PostMapping
@ResponseStatus(HttpStatus.CREATED)
public ProductDetailOutput create(@RequestBody @Valid ProductInput input) {
    UUID productId;

    try {
        productId = productManagementApplicationService.create(input);
    } catch (CategoryNotFoundException e) {
        throw new UnprocessableContentException(e.getMessage(), e);
    }

    return productQueryService.findById(productId);
}
```

Sem esse `try/catch`, criar um produto com `categoryId` inexistente devolveria **404 Not Found** — porque `CategoryNotFoundException` herda de `DomainEntityNotFoundException`. E isso estaria **errado**:

> O 404 diz "o recurso que você pediu não existe". Mas o cliente pediu `POST /products` — e `/products` existe. O que não existe é uma categoria mencionada **dentro do corpo** da requisição. Isso é conteúdo inválido: **422**.

Um cliente que recebe 404 de um `POST` conclui que o endpoint está errado e pode até parar de tentar. Recebendo 422 com `"Category with id ... was not found"`, ele entende que precisa corrigir o campo.

Note também que a causa original é preservada — `new UnprocessableContentException(e.getMessage(), e)` — mantendo a cadeia completa para o log.

---

## Tabela de mapeamento

| Exceção | Status | `type` | `detail` |
|---|---|---|---|
| `MethodArgumentNotValidException` | 400 | `/errors/invalid-fields` | genérico + `fields` com detalhe por campo |
| `DomainEntityNotFoundException` (e filhas) | 404 | `/errors/not-found` | mensagem da exceção |
| `ResourceNotFoundException` | 404 | `/errors/not-found` | mensagem da exceção |
| `DomainException` | 422 | `/errors/unprocessable-content` | mensagem da exceção |
| `UnprocessableContentException` | 422 | `/errors/unprocessable-content` | mensagem da exceção |
| `BadGatewayException` (e as duas filhas) | 502 | `/errors/bad-gateway` | dependência respondeu inválido |
| `GatewayTimeoutException` | 504 | `/errors/gateway-timeout` | dependência não respondeu a tempo |
| `Exception` (qualquer outra) | 500 | `/errors/internal` | genérico — stacktrace só no log |

Exemplo de resposta 422:

```json
{
  "type": "/errors/unprocessable-content",
  "title": "Unprocessable content",
  "status": 422,
  "detail": "Category with id 0197f2c1-3a4b-7c8d-9e0f-1a2b3c4d5e6f was not found"
}
```

---

## Como o Spring escolhe o handler

Quando várias `@ExceptionHandler` poderiam pegar a mesma exceção, o Spring escolhe a **mais específica na hierarquia**. Lançando `ProductNotFoundException`:

```
ProductNotFoundException
   └─ DomainEntityNotFoundException   ← declarada num handler ✅ vence
        └─ RuntimeException
             └─ Exception             ← também declarada, mas é mais distante
```

Por isso o `@ExceptionHandler(Exception.class)` não "engole" tudo: ele só entra quando nenhum handler mais próximo se aplica. E é justamente por isso que a ordem dos métodos no arquivo **não importa** — quem decide é a distância na hierarquia, não a posição no código.

---

## Pendências registradas

- [ ] **Bug de rota:** `ProductController` mapeia o método `disable()` em `@DeleteMapping("/{productId}/enable")` — o path diz `enable`, mas a ação desabilita. Provavelmente deveria ser `@DeleteMapping("/{productId}")` ou `.../disable`. Hoje convive com `@PutMapping("/{productId}/enable")`, que de fato habilita.
  > Na Fase 14 os contratos que testavam essa rota foram renomeados de `deleteProductByIdV1` para `disableProductV1` e apontados para `.../{id}/enable`, o que fez as **duas falhas conhecidas do `contractTest` desaparecerem**. Vale ser explícito sobre o que aconteceu ali: o contrato foi alinhado à rota, e não a rota ao contrato. A suíte ficou verde e a incoerência de nome continua exatamente onde estava — só que agora com um teste confirmando-a.
- [ ] `Collectors.toMap` sem *merge function* quebra com duas violações no mesmo campo (ver acima).
- [ ] `ResourceNotFoundException` (application) e `DomainEntityNotFoundException` (domain) se sobrepõem — depois da migração para as exceções de domínio, a primeira ficou quase sem uso e poderia ser removida.
- [ ] Não há contract test cobrindo os cenários 404 e 422 do `ProductController`.

---

## Checklist de revisão

- [ ] Exceção de domínio **não** importa nada de `org.springframework.http`
- [ ] Todo endpoint tem handler — nenhum caminho cai em página HTML de erro
- [ ] 500 nunca vaza stacktrace nem mensagem interna na resposta
- [ ] 404 usado só para "o recurso da URL não existe"; conteúdo inválido no corpo é 422
- [ ] Causa original preservada ao reempacotar exceção (`new X(msg, e)`)
- [ ] Mensagens de validação vindas de `MessageSource`, não hardcoded

---

## Referências

- [RFC 9457 — Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457)
- [Spring — Error Responses / `ProblemDetail`](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-ann-rest-exceptions.html)
- Testes de contrato desses cenários: [`stubs-contract-tests.md`](./stubs-contract-tests.md)
