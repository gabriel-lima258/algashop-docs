# Recursos `/me`: o fim do `customerId` que o cliente escolhia

> A [Gestão de usuários](./gestao-de-usuarios-e-auditoria.md) criou o primeiro `/me` e nomeou o inimigo: IDOR. Esta fase leva o padrão até o fim — carrinho no `ordering`, perfil no `authorization-server`, cartões e fatura no `billing` — e remove o `customerId` de todo path e body onde o cliente ainda podia escolhê-lo. No `product-catalog`, que não tem recurso de cliente, o endurecimento foi outro: escopo deixou de bastar nas escritas.
> Código real: `MyShoppingCartController` (ordering) · `MyUserController` (authorization-server) · `MyCreditCardController` e `MyInvoiceController` (billing) · `SecurityAnnotations` e `AuthorizationMatrixTest` nos quatro serviços.
> Conceitos em [Gestão de usuários e auditoria](./gestao-de-usuarios-e-auditoria.md) · papéis em [RBAC e controle de acesso](./rbac-e-controle-de-acesso.md) · escopos em [Resource servers e escopos](./resource-server-e-escopos.md).

---

## O bug que não precisa existir

O carrinho de compras nasceu assim:

```
POST /api/v1/shopping-carts          { "customerId": "..." }   ← o cliente diz de quem é o carrinho
GET  /api/v1/shopping-carts/{id}                               ← o cliente diz qual carrinho quer
```

Para isso ser seguro, **alguém** precisa verificar, em cada endpoint, que o id pedido pertence ao chamador. Essa verificação é código que precisa ser escrito, lembrado e mantido em *todos* os lugares — e esquecê-la em um único endpoint entre vinte é exatamente a vulnerabilidade mais comum de API REST (IDOR, *insecure direct object reference*).

A refatoração desta fase não melhora a verificação. Ela **elimina a pergunta**:

```
GET    /api/v1/customers/me/shopping-cart              ← nenhum id na URL
POST   /api/v1/customers/me/shopping-cart/items        ← nenhum customerId no body
DELETE /api/v1/customers/me/shopping-cart/items/{itemId}
```

O dono do recurso não vem da requisição. Vem do `sub` do token, que foi assinado por quem o emitiu:

```java
private ShoppingCartOutput findAuthenticateCustomerShoppingCart() {
    return forQueryShoppingCarts.findByCustomerId(securityCheck.getAuthenticatedUserId());
    //                                            ↑ o sub do JWT, nunca um parâmetro
}
```

> **Um id que o cliente não escolhe não pode ser o id de outra pessoa.** O IDOR deixa de ser um risco mitigado e vira um estado inalcançável.

O `ShoppingCartController` antigo foi **removido**, junto com o `ShoppingCartInput` que carregava `customerId` no body. O mesmo destino teve o `CreditCardController` do billing, que recebia `customerId` no path em todos os quatro endpoints.

---

## O mapa da mudança, serviço a serviço

| Serviço | Nasceu | Morreu |
|---|---|---|
| `ordering` | `MyShoppingCartController` — `/api/v1/customers/me/shopping-cart` (carrinho + itens) | `ShoppingCartController`, `ShoppingCartInput` |
| `authorization-server` | `PUT` e `DELETE` em `/api/v1/users/me` (self-update e exclusão do próprio perfil) | — (expansão do `/me` existente) |
| `billing` | `MyCreditCardController` — `/api/v1/customers/me/credit-cards` · `MyInvoiceController` — `/api/v1/customers/me/orders/{orderId}/invoice` | `CreditCardController` |
| `product-catalog` | — (não tem recurso de cliente) | a escrita autorizada só por escopo |

Repare no singular deliberado: `shopping-cart`, não `shopping-carts`. No contexto do `/me` cada cliente tem **um** carrinho — o plural do recurso administrativo não faz sentido aqui.

---

## A defesa em três camadas

A URL sem id é a primeira camada. Sozinha, ela não basta — e as outras duas é que fazem o desenho fechar.

### Camada 1 — o body também não carrega dono

Tirar o id do path e deixá-lo no body seria trocar a fechadura e deixar a janela aberta. Os inputs que o controller preenche ganharam `@JsonIgnore`:

```java
public class TokenizedCreditCardInput {
    @JsonIgnore
    private UUID customerId;      // ← preenchido pelo controller com o sub; o body não alcança

    @NotBlank
    private String tokenizedCard;
}
```

Um cliente que enviar `"customerId": "id-de-outra-pessoa"` no JSON não recebe erro — o campo é simplesmente **invisível** para a desserialização. O mesmo foi feito no `shoppingCartId` do `ShoppingCartItemInput` do ordering.

**A armadilha que este código já pisou:** o `customerId` do billing tinha `@NotNull` além do `@JsonIgnore`. A validação do `@Valid` roda **antes** do corpo do método — antes, portanto, do `input.setCustomerId(getUser())` — e o resultado era um `POST /me/credit-cards` que respondia 400 para *toda* requisição conforme o contrato. O campo que o controller preenche não pode ter constraint de presença: quando a validação olha, ele ainda não foi preenchido.

### Camada 2 — o filtro de dono vive na consulta, não depois dela

Segurança na camada de apresentação garante que o chamador está autenticado e tem o perfil certo. Ela **não** garante que o recurso consultado pertence a ele. Essa garantia foi para dentro do repositório:

```java
public interface InvoiceRepository extends JpaRepository<Invoice, UUID> {
    Optional<Invoice> findByOrderIdAndCustomerId(String orderId, UUID customerId);
    //                            ↑ o dono entra no WHERE, não num if depois do fetch
}
```

```java
@GetMapping
@CanReadMyInvoices
public InvoiceOutput findMyInvoice(@PathVariable String orderId) {
    UUID customerId = securityCheck.getAuthenticatedUserId();
    return invoiceQueryService.findByOrderIdAndCustomerId(orderId, customerId);
}
```

A consequência importa mais do que parece: a fatura de outro cliente é **indistinguível de uma fatura que não existe**. Os dois casos caem no mesmo `InvoiceNotFoundException` → 404. Um `if (!invoice.getCustomerId().equals(...))` depois do fetch, respondendo 403, confirmaria ao atacante que aquele `orderId` existe — o 404 uniforme não confirma nada. É a mesma decisão já registrada no ordering: *"para não virar oráculo de ids"*.

### Camada 3 — a anotação diz para quem o recurso existe

O `/me` é recurso de **cliente**. As anotações passaram a dizer isso:

```java
@PreAuthorize("hasAuthority('SCOPE_shopping-carts:read') and hasRole('CUSTOMER')")
public @interface CanReadMyShoppingCart {}
```

Sem o `hasRole('CUSTOMER')`, um MANAGER com o escopo certo passaria pelo `@PreAuthorize` e receberia "o carrinho do manager" — provavelmente um 404 confuso, mas conceitualmente errado: o recurso não existe para aquele público. E um token de máquina iria mais longe e quebraria feio: `getAuthenticatedUserId()` **lança** para `client_credentials`, porque máquina não tem `sub` de pessoa. A role na anotação barra os dois na porta, com o 403 que corresponde à verdade.

---

## Os três públicos, três formas de anotação

Com a fase completa, toda anotação de segurança do sistema declara **escopo e público** — e o público aparece de três formas:

| Público | Expressão | Exemplos |
|---|---|---|
| o próprio cliente | `... and hasRole('CUSTOMER')` | `CanReadMyShoppingCart`, `CanReadMyInvoices`, `CanWriteMyCreditCards`, `CanDeleteOwnProfile` |
| perfis internos (nunca cliente) | `... and not hasRole('CUSTOMER')` | `CanReadInvoices`, `CanReadShoppingCarts`, `CanWriteProducts`, `CanReadUsers` |
| só sistema (m2m) | `... and @securityCheck.isMachineAuthenticated()` | `CanWriteInvoices` (gerar fatura) |

A terceira forma merece uma pausa. Gerar fatura é um passo do checkout — **nenhum humano** deveria dispará-lo, nem o MANAGER. `hasRole` não resolve isso, porque token de máquina não tem role nenhuma; a expressão chama o bean pelo nome, e é o bean que aplica a heurística `aud` × `sub` descrita em [Gestão de usuários](./gestao-de-usuarios-e-auditoria.md).

E o `not hasRole('CUSTOMER')` da leitura geral de faturas responde à pergunta que o desenho exige: *que problema haveria se um cliente comum acessasse `GET /api/v1/orders/{orderId}/invoice`?* O endpoint recebe `orderId` livre — nas mãos de um CUSTOMER, ele seria exatamente o oráculo de faturas alheias que o `/me` existe para impedir. O endpoint continua existindo para o back-office; a role fecha a porta para quem tem alternativa própria.

### O bug que a string escondeu

A primeira versão do `CanWriteInvoices` chegou assim ao working tree:

```java
@PreAuthorize("hasAuthority('SCOPE_invoices:write') and @securityChecks.isMachineAuthenticated())")
//                                                       ↑ bean errado                            ↑ parêntese extra
```

Dois defeitos na mesma string, e **nenhum é erro de compilação**. O bean chama-se `securityCheck` (singular — `@Service("securityCheck")`), e o parêntese desbalanceado só explode quando o SpEL é avaliado: `SpelParseException` → **500** em todo `POST` de fatura. Não 403, não 401 — 500, o pior sintoma possível, porque aponta para "bug no servidor" quando a causa é um typo numa string de autorização.

É a lição da [matriz de autorização](./resource-server-e-escopos.md) cobrada de novo:

> **O que não é verificado em compilação precisa ser verificado por comportamento.** O teste que prova "máquina com o escopo → passa" foi o que transformou esse 500 em falha de suite — antes de virar incidente.

---

## O que **não** virou `/me` — e os limites do padrão

Nem toda operação do recurso antigo sobrevive à mudança de contexto:

- **Excluir o próprio perfil, só para CUSTOMER.** `@CanDeleteOwnProfile` compõe as duas condições (`@securityCheck.canAccessOwnProfile() and hasRole('CUSTOMER')`). Um MANAGER excluindo a própria conta pelo `/me` deixaria o back-office sem administrador por um clique — a exclusão de contas internas é operação administrativa, de outro MANAGER.
- **E a camada de aplicação verifica de novo.** Mesmo depois do `@PreAuthorize`, o `anonymize` do authorization-server pergunta: *o autenticado é o alvo? e ele é CUSTOMER?* A repetição não é redundância — o mesmo método serve o `DELETE /users/{id}` administrativo, e é lá que um MANAGER poderia tentar excluir a si mesmo por outra rota. A primeira versão desse guard, aliás, esquecia o token de máquina: chamava `getAuthenticatedUserId()` incondicionalmente, que lança para m2m — e o client de sistema que tinha acabado de ganhar `users:write` levaria 403 num fluxo legítimo. O guard agora libera máquina antes de perguntar identidade.
- **O self-update tem um DTO próprio.** O `PUT /users/me` recebe `MyUserUpdateInput` — só `name`. O administrativo recebe `AuthUserUpdateInput` — `name`, `type`, `enabled`. Não é o mesmo input com campos ignorados: é a afirmação, em tipo, de que o usuário não edita o próprio papel nem se auto-habilita. A sobrecarga `update(UUID, MyUserUpdateInput)` convive com a administrativa e o compilador escolhe pela assinatura.

---

## O catálogo: mesmo endurecimento, outro desenho

O `product-catalog` não tem recurso de cliente — não há `/me` a criar. Mas tinha o mesmo problema de origem: escrita autorizada só por escopo, cega para quem segura o token. A correção seguiu a mesma gramática:

```java
@PreAuthorize("hasAuthority('SCOPE_products:write') and not hasRole('CUSTOMER')")
public @interface CanWriteProducts {}

@PreAuthorize("hasAuthority('SCOPE_products:stock:write') and hasRole('MANAGER')")
public @interface CanWriteProductsStock {}
```

A segunda linha é a mais restritiva do sistema: estoque é operação de **MANAGER humano**. Máquina fica fora *por construção* — `client_credentials` não carrega o claim `role`, logo `hasRole('MANAGER')` é falso para qualquer m2m — e OPERATOR fica fora por política, a mesma da tabela `auth_user_type_client_scope` do authorization server. As leituras seguem só com escopo, porque o catálogo é lido por clients de máquina (`ecommerce-m2m`, `ordering`) que nunca terão role.

---

## Como os testes travam tudo isso

Cada camada nova ganhou o teste que a percebe quebrar:

**As matrizes de autorização** deixaram de ter três colunas (sem token / escopo errado / escopo certo) e ganharam **grupos por público**. A do catálogo é o exemplo mais claro — 86 casos em três grupos:

```
leitura   → escopo sozinho passa (m2m precisa continuar lendo)
escrita   → CUSTOMER com o escopo certo → 403   ← a regra nova central
estoque   → máquina, OPERATOR e CUSTOMER → 403; só MANAGER passa
```

A do billing prova os três públicos de fatura (humano gerando fatura → 403 *mesmo com o escopo*; máquina → passa), e o authorization-server ganhou sua primeira matriz — importando a filter chain real e a implementação real do bean `securityCheck`, para que o SpEL avalie contra tokens sintéticos com `aud`/`sub` de verdade.

**Os contratos acompanharam a API.** Os `.groovy` do carrinho e dos usuários foram reescritos para os paths `/me` — e o `createShoppingCartV1` com `customerId` no body **deixou de existir**, porque o contrato que o consumidor conhece também não pode mais oferecer o campo. As bases (`ShoppingCartBase`, `MyUserBase`) mockam o `SecurityCheckApplicationService` com um `sub` fixo, o mesmo padrão do `OrderBase`.

**Os ITs provam o isolamento com dados reais.** O `MyShoppingCartControllerIT` verifica que o carrinho devolvido é sempre o do `sub` do token — nunca o do outro cliente que o seed planta de propósito. O `InvoiceQueryServiceIT` faz a prova da camada 2: mesma fatura, outro `customerId` → `InvoiceNotFoundException`.

> **Um teste de segurança não afirma "é 200"; afirma "não parou em 401/403".** Acoplar a matriz ao resultado de negócio a faria falhar por motivo errado — a convenção vem da primeira matriz e sobreviveu a todas.

---

## Armadilhas

- **`@NotNull` em campo que o controller preenche** — a validação roda antes do set; o endpoint morre com 400 permanente. `@JsonIgnore` sim, constraint de presença não.
- **Nome de bean em SpEL é contrato** — `@securityChecks` compilou e quebrou em runtime. O plural custou um 500.
- **Parêntese em string SpEL** — o parser só roda na primeira requisição. A matriz é o único lugar que percebe antes.
- **`hasRole` para máquina é sempre falso** — não use role para *incluir* m2m; use `@securityCheck.isMachineAuthenticated()`. E lembre que `getAuthenticatedUserId()` **lança** para máquina: todo guard de camada de aplicação precisa perguntar `isMachineAuthenticated()` primeiro.
- **403 no lugar de 404 confirma existência** — o filtro de dono pertence ao `WHERE` da consulta, e o "não é seu" deve ser indistinguível do "não existe".
- **Recurso `/me` no singular** — `/customers/me/shopping-cart`, porque há um por cliente. Copiar o plural do recurso administrativo trai o modelo.

## Pendências registradas

- [ ] **O webhook do FastPay continua público e sem verificação de assinatura** — muda estado de fatura; público é necessário, sem verificação não deveria ser.
- [ ] **`algashop-test` ainda declara `products:stock:write` no YAML** do authorization server, mas o escopo virou letra morta: m2m leva 403 no estoque. Remover é higiene pendente.
- [ ] **As specs OpenAPI não documentam a dimensão de segurança** — `product-catalog.yml` não tem `securitySchemes` nem 401/403; as demais merecem conferência.
- [ ] **`isMachineAuthenticated()` segue heurística** (`aud` × `sub`) — e agora decide quem gera fatura. O claim explícito continua sendo o caminho robusto.
- [ ] **O billing não tem Spring Cloud Contract** — a cobertura de contrato é só matriz + IT, diferente dos vizinhos.

## Checklist de revisão

- [ ] O endpoint novo recebe id de dono por path ou body? Se sim, por que não é `/me`?
- [ ] O input tem campo que o controller preenche? Então `@JsonIgnore` e **sem** constraint de presença.
- [ ] A consulta filtra por dono no repositório, ou verifica depois do fetch?
- [ ] Recurso alheio responde 404 ou um 403 que confirma existência?
- [ ] A anotação declara escopo **e** público? Qual dos três: CUSTOMER, interno, máquina?
- [ ] A matriz tem um caso que falharia se a role sumisse da expressão?
- [ ] O guard de aplicação pergunta `isMachineAuthenticated()` antes de pedir identidade?

## Referências

- [OWASP — Insecure Direct Object Reference Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Insecure_Direct_Object_Reference_Prevention_Cheat_Sheet.html)
- [OWASP API Security Top 10 — API1: Broken Object Level Authorization](https://owasp.org/API-Security/editions/2023/en/0xa1-broken-object-level-authorization/)
- [Spring Security — Method Security](https://docs.spring.io/spring-security/reference/servlet/authorization/method-security.html)
- [Gestão de usuários e auditoria](./gestao-de-usuarios-e-auditoria.md) · [RBAC e controle de acesso](./rbac-e-controle-de-acesso.md) · [Resource servers e escopos](./resource-server-e-escopos.md)
