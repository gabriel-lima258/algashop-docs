# Linha do tempo

> A jornada de aprendizado em ordem cronológica — **o que** foi construído e, principalmente, **por que** naquele momento.
> Reconstruída a partir do histórico Git dos cinco repositórios.

---

## Fase 1 — DDD tático no `ordering` (fev/2026)

O projeto começa pelo domínio mais rico, e de dentro para fora: **nada de banco, nada de HTTP** nas primeiras semanas.

| Data | Marco | O que se aprende |
|---|---|---|
| 11/02 | Entity model pattern | Entidade tem **identidade**, não é struct de dados |
| 12–15/02 | Value objects + testes unitários | `Money`, `Email`, `Quantity`… objetos imutáveis definidos pelo **valor** |
| 16/02 | Aggregate root e factories | Quem é a fronteira de consistência; quem pode ser referenciado de fora |

**A lição da fase:** modelar o domínio antes de escolher tecnologia. O agregado `Order` foi escrito, testado e validado sem que existisse uma linha de JPA. Se o domínio só faz sentido com o banco por perto, o modelo provavelmente é anêmico.

---

## Fase 2 — Persistência e serviços (fev/2026)

Só depois de o domínio estar de pé é que a infraestrutura entra.

| Data | Marco | O que se aprende |
|---|---|---|
| 21/02 | Repositórios JPA de `Order` e `Customer` | Repositório é abstração **de domínio**, implementada na infra |
| 22/02 | Persistência do `ShoppingCart` | Cobertura completa antes de seguir |
| 22/02 | Domain services | Onde mora a regra que não pertence a **nenhum** agregado |
| 26–27/02 | Application services + testes de integração | Caso de uso orquestra; não contém regra de negócio |
| 27/02 | Domain events no `Order` | Desacoplar efeitos colaterais do fluxo principal |

**A lição da fase:** a distinção entre *application service* (orquestra, abre transação, converte DTO) e *domain service* (regra que envolve mais de um agregado). Misturar os dois é o começo do domínio anêmico.

---

## Fase 3 — Padrões de leitura e consulta (mar/2026)

| Data | Marco | O que se aprende |
|---|---|---|
| 01/03 | Specification pattern | Regra de negócio como objeto combinável, em vez de `if` aninhado |
| 02/03 | CQRS com Criteria API | Consulta não passa pelo agregado — projeta direto |
| 02/03 | Paginação + `PageModel` | Nunca devolver lista inteira; contrato de página próprio |

**A lição da fase:** carregar um agregado inteiro para exibir uma lista é desperdício. Escrita precisa do agregado (para proteger invariantes); leitura não precisa de nada além dos campos que a tela mostra.

> [`specification.md`](../01-arquitetura-design/specification.md) · [`cqrs.md`](../01-arquitetura-design/cqrs.md) · [`paginacao.md`](../02-persistencia/paginacao.md)

---

## Fase 4 — A API REST e os contratos (mar/2026)

| Data | Marco | O que se aprende |
|---|---|---|
| 08/03 | `CustomerController` + contract tests | Contrato **antes** da implementação |
| 08–09/03 | Nasce o `product-catalog` | Segundo serviço; contract-driven desde o dia um |
| 03/03 | Nasce o `billing` | Terceiro serviço |
| 04/03 | Correções de validação no `billing` | Test data builders com bug encontrado pelos testes |
| 20/03 | REST Docs, WireMock, stubs | Documentação gerada **a partir dos testes** |

**A lição da fase:** com mais de um serviço, testar integração subindo tudo vira inviável. O contrato publicado por um serviço vira o stub que o outro consome nos testes — cada um roda isolado, e o contrato quebrado aparece no CI.

> [`stubs-contract-tests.md`](../03-testes-integracao/stubs-contract-tests.md)

---

## Fase 5 — Banco de verdade e ambiente (mar/2026)

| Data | Marco | O que se aprende |
|---|---|---|
| 22/03 | Flyway no `ordering` e no `billing` | Schema versionado como código |
| 22/03 | PostgreSQL + isolamento de dados de teste | Bancos `*_test` separados |
| 22/03 | Infra de gateway de pagamento no `billing` | Abstração antes da integração real |

**A lição da fase:** `ddl-auto: update` funciona no seu notebook e falha em produção. Migration versionada é a única forma de o schema evoluir de maneira previsível em todos os ambientes.

> [`flyway.md`](../02-persistencia/flyway.md)

---

## Fase 6 — Integração externa e containers (abr/2026)

| Data | Marco | O que se aprende |
|---|---|---|
| 15/04 | Integração FastPay no `billing` | Chamar API externa de verdade: timeout, erro, recusa |
| 15/04 | Cartão de crédito no checkout do `ordering` | Fluxo ponta a ponta atravessando dois serviços |
| 15/04 | Dockerfiles + configuração por perfil | Mesma imagem, ambientes diferentes |
| 15/04 | Nasce o `billing-scheduler` | Job agendado separado da API |

**A lição da fase:** o que passa pela rede falha. Integração externa fica atrás de uma abstração, com implementação fake para desenvolvimento e testes.

> [`docker.md`](../04-infraestrutura/docker.md) · [`scheduled-jobs.md`](../04-infraestrutura/scheduled-jobs.md)

---

## Fase 7 — Refatoração arquitetural (jun/2026)

| Data | Marco | O que se aprende |
|---|---|---|
| 22/06 | `ordering` reestruturado para hexagonal | `core/ports/in` e `core/ports/out`, adapters na infra |
| 22/06 | Testcontainers no `billing` | Banco real e efêmero nos testes, sem infra compartilhada |
| 22/06 | `billing-scheduler` implementado | Cancelamento de faturas expiradas |

**A lição da fase:** a refatoração veio **depois** de o domínio estar consolidado e coberto por testes. Arquitetura hexagonal aplicada cedo demais vira cerimônia sem propósito; aplicada quando as dependências já incomodam, resolve um problema real. Os testes é que tornaram a reestruturação segura.

> [`ports-hexagonal.md`](../01-arquitetura-design/ports-hexagonal.md)

---

## Fase 8 — NoSQL no `product-catalog` (jul/2026)

| Data | Marco | O que se aprende |
|---|---|---|
| 14/07 | `MongoConfig` — UUID e conversores | Representação de UUID, `OffsetDateTime` ↔ `Date` |
| — | Agregado `Product` com `@Document` | Modelagem documental vs. relacional |
| — | `@DocumentReference` para categoria | Embutir vs. referenciar *(revertido na Fase 12)* |
| — | Auditoria + `@Version` | `@CreatedDate`/`@CreatedBy`, lock otimista |
| — | Hierarquia de exceções de domínio | Domínio não conhece HTTP; 404 vs. 422 |
| — | `ProductSummaryOutput` + conversores do ModelMapper | Projeção de listagem, slug, descrição abreviada |

**A lição da fase:** persistência poliglota escolhida pelo padrão de acesso, não por preferência. E a percepção de que o **mesmo desenho de domínio** — agregado protegendo invariantes, regra no setter, campo derivado calculado — funciona igual em documento e em tabela. O que muda é só o adapter.

> [`product-catalog-mongo.md`](../02-persistencia/product-catalog-mongo.md) · [`tratamento-erros-api.md`](../03-testes-integracao/tratamento-erros-api.md)

---

## Fase 9 — Consultas dinâmicas e carga de dados (jul/2026)

Com o agregado persistindo, o problema vira **ler**: uma listagem com nove filtros opcionais, e um banco de desenvolvimento que nasce vazio.

| Marco | O que se aprende |
|---|---|
| `Query` + `Criteria` no `ProductQueryServiceImpl` | Montar a consulta em tempo de execução, em vez de um método de repositório por combinação |
| Hierarquia `PageFilter` → `SortablePageFilter<T>` → `ProductFilter` | Genérico amarrado a enum torna a ordenação type-safe |
| `ProductFilter` como command object | O Spring binda a query string inteira num objeto — filtro novo não muda assinatura nenhuma |
| `$expr` no filtro `hasDiscount` | Comparar dois campos do **mesmo** documento; e por que isso não usa índice |
| Busca textual por regex com `$or` | Uma alternativa ao índice de texto, com os trade-offs explícitos |
| `count` antes de `skip`/`limit` | A ordem que faz o `totalPages` sair certo |
| `ProductNameProjection` + `fields` | Projeção no servidor, não na aplicação |
| `DataLoader` + Extended JSON | Substituto do Flyway num banco sem migration |
| Tasks `test` / `contractTest` / `integrationTest` | Teste rápido não pode depender de container de pé |

**A lição da fase:** o filtro dinâmico é o mesmo padrão da Criteria API do `ordering` — muda a sintaxe, não a ideia. E surgiu de novo a pergunta de sempre: calcular na consulta (`$expr`, flexível, sem índice) ou gravar o campo já calculado na escrita (rápido, mas duplica estado). O agregado já tinha escolhido o segundo caminho para o `discountPercentageRounded`; o filtro escolheu o primeiro para o `hasDiscount`. Vale saber que a incoerência existe.

> [`consultas-mongo-criteria.md`](../02-persistencia/consultas-mongo-criteria.md) · [`carga-de-dados-mongo.md`](../04-infraestrutura/carga-de-dados-mongo.md)

---

## Fase 10 — Índices e busca textual (jul/2026)

A fase anterior terminou com uma pendência escrita em letras grandes: *nenhum índice foi criado para os campos filtrados*. Esta fase é a resposta — e começa por um problema de método: **com dezenas de documentos não há o que medir**.

| Marco | O que se aprende |
|---|---|
| `products-large.json` com 560 mil documentos | Otimização sem massa é palpite; primeiro se cria o cenário onde a diferença aparece |
| `explain("executionStats")` | `COLLSCAN` vs. `IXSCAN`, e os números que dizem se o índice foi mesmo usado |
| `@CompoundIndex` e a regra **ESR** | A ordem dos campos no índice não é estética — igualdade, ordenação, faixa |
| Dois índices compostos, não um | Um índice serve bem **uma** ponta por consulta; a outra cairia em ordenação em memória |
| `partialFilter: {enabled: true}` | Índice menor, em troca de só ser usado quando a consulta traz o predicado explícito |
| `@TextIndexed` + `$text` no lugar do `$or` de regex | Índice de texto é único por coleção, faz stemming e casa palavra inteira |
| `@TextScore` e `Sort.by("score")` | Ordenar por relevância; um campo que não existe no documento |
| `drop()` → `deleteMany({})` no `DataLoader` | Ordem dos eventos na subida: os índices já existiam quando a carga rodava |
| `CategoryQueryServiceImpl` implementado | O filtro dinâmico da fase 9 aplicado uma segunda vez, quase de graça |

**A lição da fase:** índice não é melhoria, é **troca** — acelera leitura, encarece escrita e ocupa memória, e índice que ninguém usa é só custo (o `brand` desta fase é justamente esse caso). A outra lição é mais desconfortável: quase todo ganho veio com uma perda ao lado. O `$text` trouxe velocidade e tirou a busca por substring e por marca; o `partialFilter` encolheu o índice e o tornou inútil para quem não filtra por ativo. Escolher bem depende de saber qual consulta realmente acontece — e disso o `explain` é a única fonte honesta.

> [`indices-mongo.md`](../02-persistencia/indices-mongo.md)

---

## Fase 11 — Aggregation pipeline (jul/2026)

A listagem estava rápida, mas ainda mentia sobre o custo: para mostrar o nome da categoria, cada produto disparava uma leitura extra. Era o **N+1** registrado como pendência desde a Fase 8 — e resolvê-lo, *pela ferramenta de consulta*, exigiu trocar `find()` por pipeline. A Fase 12 mostraria que o problema era outro.

| Marco | O que se aprende |
|---|---|
| `find()` → `aggregate()` na listagem | Pipeline como esteira de estágios, cada um alimentando o próximo |
| `$lookup` + `$unwind` | O join que acaba com o N+1 — e que `unwind` sem flag é **inner join** *(aposentados na Fase 12)* |
| `$project` com `andExpression` | `hasDiscount`, `inStock` e `shortDescription` calculados no banco, não no ModelMapper |
| DTO como destino do `aggregate` | O `$project` devolve no formato do DTO, e o `TypeMap` do ModelMapper some |
| `$addFields` com `$meta: textScore` | `@TextScore` **não** funciona em pipeline; o campo tem que ser criado à mão |
| `AggregationOperation` como lambda | Escapatória para qualquer estágio que o Spring Data não embrulhou |
| `Criteria` acumulado em lista + `andOperator` | Um `$match` recebe **um** criteria, não uma coleção deles |
| `$expr` escrito como criteria comum | `AggregationExpressionCriteria` é `CriteriaDefinition`, mas não `Criteria` |
| A ordem dos estágios | Só o primeiro `$match` usa índice; tudo depois é memória |

**A lição da fase:** a mesma pergunta das fases anteriores voltou de outro jeito — **calcular onde?** Antes era "no banco ou na escrita"; aqui é "no banco ou na aplicação". A resposta acabou sendo *depende do campo*: `hasDiscount` e `inStock` foram para o `$project` porque são comparações triviais que o Mongo faz de graça; o `slug` ficou em Java porque tirar acento com operador do Mongo custaria uma cadeia de `$replaceAll`. Poder empurrar tudo para o banco não é motivo para empurrar.

A segunda lição é sobre controle: no `find()` o Mongo decide como executar; num pipeline **a ordem escrita é a ordem executada**, sem otimizador reescrevendo nada. Ganha-se precisão e herda-se a responsabilidade — o pipeline juntava e projetava antes de paginar, e isso ficou documentado como o que ainda faltava arrumar.

> [`agregacoes-mongo.md`](../02-persistencia/agregacoes-mongo.md)

---

## Fase 12 — Desnormalização e eventos (ago/2026)

A Fase 11 tinha resolvido o N+1 com um `$lookup`. Esta fase pergunta por que ele existia: se **toda** listagem precisa do nome da categoria, esse nome deveria estar no documento. A categoria virou uma cópia embutida, o join sumiu — e apareceu um problema novo, que é o assunto da segunda metade da fase: **cópia envelhece**.

| Marco | O que se aprende |
|---|---|
| `@DocumentReference` → `ProductCategory` embutido | Normalizado × desnormalizado, e que a decisão da Fase 8 estava do lado errado da própria regra |
| `$lookup` + `$unwind` aposentados | A justificativa de uma ferramenta pode evaporar sem o código quebrar |
| `'category.id'` virando `category._id` | Propriedade chamada `id`, mesmo embutida, vira `_id` — e só o `getIndexes()` conta a verdade |
| `Product extends AbstractAggregateRoot` | `registerEvent` **enfileira**; quem publica é o `save()` do repositório |
| Cinco eventos de domínio | Fato só é fato quando o estado muda — daí as guardas do `setEnabled` e do `changePrice` |
| `ApplicationMessagePublisher` | Porta de saída: a `application` publica sem conhecer o Spring |
| `@EventListener` + `@Async` + `updateMulti` | Consistência eventual na prática, e tudo que falta para virar mensageria |
| `validatePrices` no construtor e no `changePrice` | Invariante que depende de dois campos não cabe num setter |
| `_class` apontando para pacote inexistente | Falha silenciosa: o Spring Data engole e cai no tipo alvo |

**A lição da fase:** as Fases 8, 11 e 12 são três respostas para a **mesma** pergunta — como mostrar o nome da categoria numa listagem — e a terceira só apareceu quando a pergunta mudou de camada. A Fase 11 otimizou a consulta; a Fase 12 mudou a modelagem e a consulta ficou trivial. **Otimizar uma consulta é, às vezes, adiar a decisão de modelagem.**

A segunda lição é que desnormalizar não é atalho, é dívida com prazo: a leitura ficou barata às custas da escrita, e a diferença entre "cópia" e "bagunça" é ter um dono declarado e um mecanismo explícito de propagação. Aqui o dono é a coleção `categories` e o mecanismo é um evento assíncrono — com todas as garantias que ele **não** dá registradas em letras grandes.

E a terceira, involuntária: mover um invariante é onde ele se perde. A regra de preço saiu dos setters para o `changePrice` e chegou lá comparando o valor novo com o antigo, aceitando promoção mais cara que o preço cheio. Um teste de agregado puro pegou o que a leitura não pegou.

> [`desnormalizacao-mongo.md`](../02-persistencia/desnormalizacao-mongo.md) · [`eventos-e-listeners.md`](../01-arquitetura-design/eventos-e-listeners.md)

---

## Fase 13 — Concorrência e atomicidade no estoque (ago/2026) ← etapa atual

Uma pendência aberta desde a Fase 8 dizia que `quantityInStock` não tinha operação pública de entrada ou saída — produto criado pela API ficava travado em `0`. Resolvê-la parecia trivial: um `withdraw()` no agregado. A fase escolheu o caminho difícil, e é aí que está a lição: **carregar o agregado para alterar estoque é exatamente o que não se pode fazer**, porque entre ler, conferir e gravar cabe a requisição inteira de outra pessoa.

| Marco | O que se aprende |
|---|---|
| `findAndModify` com a regra no filtro | Atualização condicional atômica — o banco previne o conflito em vez de detectá-lo |
| `$inc` em vez de `set` | Delta compõe, valor absoluto sobrescreve: dois `$inc` concorrentes somam, dois `set` se atropelam |
| `gte(quantity)` no filtro | `$inc` sozinho não impede estoque negativo; quem impede é a condição de casamento |
| `returnNew(false)` | "Atômico" não basta se o valor anterior vier de outra ida ao banco |
| `Result` com antes **e** depois | Evento nasce de **transição**, não de estado — daí `isOutOfStock` exigir `previous != 0` |
| `StockService` como serviço de domínio | Comportamento de negócio que não cabe em um agregado, porque não passa por ele |
| `DomainEventPublisher` | Sem `save()`, o `AbstractAggregateRoot` não publica nada — o evento precisa de outra porta |
| `@Version` incrementado à mão | O Spring Data faria sozinho; a linha explícita é o que o faz pular a dele |
| Banco `product_catalog_test` | `auto-drop: true` apaga coleções — a suíte não pode apontar para o banco de desenvolvimento |

**A lição da fase:** as três estratégias de concorrência resolvem problemas diferentes, e a escolha não é sobre qual é mais forte. Lock otimista **detecta** o conflito e devolve a decisão para alguém; atualização condicional **previne**, e não há decisão a devolver. Saque de estoque é "subtraia 2", não "passe a valer 40" — e operações que compõem não precisam de ninguém para arbitrar. Escolher a estratégia é escolher quem fica com o problema.

A segunda lição é sobre o que um teste prova. Os testes sequenciais do ajuste passavam em qualquer implementação, inclusive na errada; só um teste com threads de verdade — soltas juntas por um `CountDownLatch` — distingue "a conta fecha" de "a conta fecha sob concorrência". **Teste que não expõe a condição de corrida não testa a garantia que interessa.**

E a terceira, que voltou pela segunda vez seguida: mover uma responsabilidade é onde ela se perde. A leitura do valor anterior estava *fora* da operação atômica, e o resultado era evento duplicado — a operação estava certa, o entorno dela não.

> [`concorrencia-e-atomicidade.md`](../02-persistencia/concorrencia-e-atomicidade.md) · [`eventos-e-listeners.md`](../01-arquitetura-design/eventos-e-listeners.md)

---

## Onde o projeto está

**Consolidado:**
- DDD tático completo no `ordering` (agregados, VOs, eventos, specifications)
- CQRS com projeções e paginação
- Contract-driven development entre serviços
- Flyway, Docker, perfis por ambiente
- Arquitetura hexagonal no `ordering`
- Modelagem documental no `product-catalog`
- Consulta dinâmica paginada no `product-catalog`
- Índices e busca textual no `product-catalog`
- Aggregation pipeline na listagem — campos derivados no servidor
- Categoria desnormalizada no produto, com propagação por evento assíncrono
- Eventos de domínio no `Product` via `AbstractAggregateRoot`
- Estoque com atualização condicional atômica, coberto por teste de concorrência

**Próximos passos naturais:**
- Mensageria real entre serviços (hoje os eventos são internos ao processo, aqui e no `ordering`)
- Retentativa, dead letter e reconciliação para a propagação da categoria e para os eventos de estoque
- Contratos Spring Cloud Contract para `/restock` e `/withdraw`
- Reordenar os estágios do pipeline: paginar **antes** do `$project`
- Índices na coleção `categories` (hoje a busca por nome é varredura com regex)
- Devolver `brand` à busca por termo (`@TextIndexed`) e tirar o índice que ficou órfão
- Testes do pipeline e um que afirme `IXSCAN` no `explain` (o agregado `Product` já tem o seu)
- Autenticação (a auditoria usa um `UUID` aleatório como placeholder)
- Observabilidade — tracing distribuído, métricas, log estruturado
- Resiliência — circuit breaker e retry nas chamadas entre serviços

---

## O padrão que se repete

Olhando as treze fases em conjunto, a ordem foi sempre a mesma:

```
domínio → testes → persistência → API → contrato → infraestrutura → refatoração
```

Nunca o contrário. A tecnologia entrou depois que o problema estava entendido, e cada refatoração grande só aconteceu com rede de testes por baixo.
