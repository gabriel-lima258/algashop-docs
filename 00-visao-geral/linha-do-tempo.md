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
| `products-large.json` com 560 mil documentos | Otimização sem massa é palpite; primeiro se cria o cenário onde a diferença aparece *(o arquivo foi removido do repositório na Fase 19)* |
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

## Fase 13 — Concorrência e atomicidade no estoque (ago/2026)

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

## Fase 14 — Replica set, transações e histórico (ago/2026)

A Fase 13 deixou o saldo correto e **sem memória**: o estoque dizia `40` e nada explicava como saiu de `50`. A resposta — uma coleção de movimentações — criou um problema que não existia: agora são **duas** escritas, em coleções diferentes, e o Mongo só garante atomicidade de uma por vez.

| Marco | O que se aprende |
|---|---|
| Mongo de nó único → replica set `rs0` | Transação no Mongo **não existe** fora de um replica set — é o oplog que a sustenta |
| `MongoTransactionManager` | Sem o bean, `@Transactional` não falha: ele não faz nada |
| `@Transactional` só em `restock`/`withdraw` | Escrita de **um** documento já é atômica; transação ali seria custo sem benefício |
| `findAndModify` entrando na transação sem mudar de código | Depender de `MongoOperations`, e não do driver, fez o comportamento novo chegar por baixo |
| `StockMovement` sem `@Version` nem `AbstractAggregateRoot` | Registro imutável de fato consumado não tem invariante para proteger |
| Listeners síncronos dentro do commit | O evento deixou de ser um "depois" — e um `@Async` o tiraria do rollback em silêncio |
| `priority: 0` nos secundários | Failover desligado **de propósito**: aqui o cluster existe para habilitar transação, não para disponibilidade |
| `withReplicaSet()` no Testcontainers | No 1.x era padrão; no 2.x virou opt-in, e sem ele o teste de transação nem sobe |
| Três testes de concorrência mortos por uma mudança de compose | Infraestrutura que muda invalida configuração de teste **sem avisar** |
| `response` aninhado dentro de `request` | Dois contratos verdes que não verificavam resposta nenhuma |

**A lição da fase:** um requisito de domínio atravessou o sistema inteiro até a infraestrutura. "Quero saber como o estoque chegou aqui" virou uma coleção nova, virou transação, e transação virou três containers no `docker-compose`. Não foi escolha de tecnologia procurando problema — foi a única forma de atender o requisito, e o caminho inverso do que se costuma ver. **Às vezes a camada mais baixa muda porque a mais alta pediu.**

A segunda lição é sobre falha silenciosa, e ela apareceu três vezes na mesma leva: `@Transactional` sem gerenciador não avisa; `@Async` saindo do rollback não avisa; `response` dentro de `request` não avisa. As três passam em revisão de código porque **o código está sintaticamente perfeito** — o que falta é uma garantia que nada declara e nada verifica. Só teste que já foi visto falhando distingue esses casos de código correto.

E a terceira: as credenciais que sumiram do compose mataram os três testes de concorrência da fase anterior, e ninguém notou porque a suíte de integração não chegou a ser executada depois da mudança. **A rede de testes só protege se rodar.**

> [`transacoes-mongo.md`](../02-persistencia/transacoes-mongo.md) · [`ambiente-local.md`](../04-infraestrutura/ambiente-local.md) · [`stubs-contract-tests.md`](../03-testes-integracao/stubs-contract-tests.md)

---

## Fase 15 — Cache com Redis, dos dois lados (ago/2026)

Até aqui o catálogo respondia sempre indo ao Mongo. Esta fase põe cache — e o que a torna interessante é que ele entrou nos **dois lados da mesma chamada**: o `product-catalog` cacheia as próprias respostas, e o `ordering` cacheia o que recebe do `product-catalog`. Somando os cabeçalhos HTTP, o mesmo produto passa a existir em quatro lugares.

| Marco | O que se aprende |
|---|---|
| `@Cacheable` × `@CachePut` | Cache-aside popula na leitura; write-through popula na escrita |
| `create` devolvendo `ProductDetailOutput` | `@CachePut` cacheia o **retorno** — uma decisão de cache mudou a assinatura de um método |
| `@Cacheable` na interface do `@HttpExchange` | O proxy do cache envelopa o proxy HTTP: dá para cachear uma chamada de rede sem tocar em quem a faz |
| Client-side sem `@CacheEvict` | Quem cacheia dado dos outros não fica sabendo quando ele muda — sobra o TTL, e não por escolha |
| `ETag` derivado do `@Version` | O validador certo já existia no documento; hash do corpo daria o mesmo por mais trabalho |
| `checkNotModified` → `304` | Resposta sem corpo, e a consulta ao banco nem acontece |
| `isCacheable()` só no filtro default | Cachear filtro livre é armadilha de cardinalidade: chaves demais, reuso nenhum |
| `implements Serializable` transitivo | O serializador default é o do Java, não JSON — e a exigência contamina todos os campos |
| `CacheErrorHandler` fail-open | Cache fora do ar tem que significar "vou à origem", não "desisti" |
| `--requirepass ${REDIS_PASSWORD}` vazio | `${VAR}` no compose vem do `.env`, nunca do `environment:` do próprio serviço |
| `--appendonly no` + `allkeys-lru` | É cache, não banco: perder tudo custa misses, não dado |

**A lição da fase:** cache mal configurado **não quebra nada**. O Redis subiu sem senha, as duas aplicações mandavam senha, a conexão falhava, o `CacheErrorHandler` engolia — e tudo respondia certo, o tempo todo, indo ao banco. O `DBSIZE` ficou em zero por horas sem um alerta. É o oposto de um banco mal configurado, que falha alto e imediatamente. **A evidência de que um cache funciona não é o serviço responder; é haver chave nele.**

A segunda lição é sobre a soma das camadas. Cada TTL parece uma decisão local — um minuto aqui, cinco ali —, mas o dado que chega ao usuário tem a idade **somada**: Redis do catálogo, mais Redis do ordering, mais o `max-age` do navegador. Quem configura uma camada sem olhar as outras acha que está entregando frescor de um minuto e está entregando o de sete.

E a terceira, que veio de graça: `restock` e `withdraw` não carregam nem salvam o produto — ajustam o estoque direto no banco. Nada neles *parece* mexer no agregado, e por isso foram os últimos a receber `@CacheEvict`. **O método que muda estado sem tocar no objeto é o que mais facilmente esquece o cache.**

> [`cache.md`](../01-arquitetura-design/cache.md) · [`redis.md`](../04-infraestrutura/redis.md)

---

## Fase 16 — Resiliência entre serviços (ago/2026)

Até aqui as chamadas entre serviços eram otimistas: nenhuma tinha timeout, e uma dependência lenta prendia a thread até desistir — o que nunca acontecia. Esta fase coloca as cinco proteções clássicas. O que a torna rica é que **o `ordering` e o `billing` chegaram a decisões opostas** com a mesma biblioteca, na mesma leva.

| Marco | O que se aprende |
|---|---|
| Timeout no `RestClient` | É o padrão que **habilita** os outros: sem ele o circuito nunca abre, porque só reage a falhas que terminaram |
| `RetryPolicy.includes(...)` | Retentar 4xx é desperdício garantido; a lista do que repetir é decisão de negócio |
| Backoff exponencial | Retentar imediatamente soma carga a quem já está mal |
| `@ConcurrencyLimit` | Bulkhead **bloqueante**: a thread excedente espera, não falha rápido |
| Circuito **programático**, sem anotação | E sem limiar de falhas: **uma** exceção leva CLOSED → OPEN |
| Fallback no frete, nenhum no produto | A pergunta não é "dá para ter fallback", é "existe resposta aproximada **aceitável**" |
| `fastpayCB` × `fastpayWriteCB` | Dois circuitos para o **mesmo host**, separados por idempotência |
| `maxRetries(0)` no `capture` | Timeout num POST de cobrança não cancela nada do outro lado — repetir é cobrança dupla |
| `InvoicePaymentTransactions` | Chamada HTTP fora de transação: nenhum breaker corrige pool de conexões esgotado |
| Clients movidos para `adapters/out` | O doc de hexagonal sempre descreveu assim; agora o código concorda |
| `fault: EMPTY_RESPONSE` no WireMock | Caos no stub, não no controller — o stub roda em CI |

**A lição da fase:** os cinco padrões não são cinco escolhas independentes — eles se aninham, e a ordem é o que os torna úteis. Timeout por dentro de retry, retry por dentro do circuito, circuito por dentro do bulkhead. Cada um sozinho protege pouco: **timeout sem circuito faz o serviço morrer devagar em vez de rápido; circuito sem timeout nunca abre.**

A segunda lição é a assimetria dos dois clients. O mesmo desenvolvedor, na mesma tarde, deu fallback ao frete e negou ao produto — e estava certo nas duas vezes. Frete estimado por baixo custa margem; preço de produto inventado custa a venda e a confiança. **A pergunta que decide um fallback é do negócio, não da engenharia** — e o custo de acertar essa pergunta é que o sistema passa a mentir em silêncio quando a resposta é "sim".

E a terceira, que veio de graça e é a mais transferível: a mudança mais importante do `billing` **não é nenhum dos cinco padrões**. É ter tirado a chamada HTTP de dentro da transação. Nenhum circuit breaker corrige um pool de conexões esgotado, porque quando a chamada chega ao breaker o recurso já foi tomado acima dele. **Resiliência não é uma camada que se adiciona por cima — é uma propriedade de como recursos escassos são adquiridos.**

> [`resiliencia.md`](../01-arquitetura-design/resiliencia.md) · [`resiliencia-config.md`](../04-infraestrutura/resiliencia-config.md)

---

## Fase 17 — Health check, readiness e degradação (ago/2026)

A Fase 16 protegeu as chamadas e deixou uma pendência: **ninguém sabia se um circuito tinha aberto** — o estado só existia em `log.info`. Esta fase expõe isso, e no caminho descobre que a pergunta "o serviço está bem?" é mal formulada.

| Marco | O que se aprende |
|---|---|
| Liveness × readiness | Reiniciar não conserta dependência externa; tirar de rotação, sim |
| Status `DEGRADED` inventado | `Status` não é enum — qualquer string vira status, e o `status.order` é que lhe dá significado |
| `readiness` só com o banco | Cache e circuito fora **não** tiram a instância de rotação; foi verificado |
| Indicador nativo do Redis desligado | Ele reportaria `DOWN`; para cache isso é grave demais |
| `@Component("cache")` | O nome do bean é o nome no endpoint — renomear a classe muda o contrato |
| `discovery.client.health-indicator: false` | Uma dependência transitiva registrava um indicador `UNKNOWN` eterno que envenenava o agregado |
| `DEGRADED` devolvendo **HTTP 200** | Só `DOWN` e `OUT_OF_SERVICE` viram 503 — para um probe, degradado e saudável são iguais |
| Dockerfile do `product-catalog` | O último serviço sem imagem passou a ter uma, e entrou no compose |

**A lição da fase:** o vocabulário de estados é uma decisão de arquitetura, não de configuração. `UP` e `DOWN` só têm dois valores porque a maioria dos sistemas nunca precisou de um terceiro — e o terceiro é justamente o caso mais comum em microsserviços: **uma dependência caiu e o serviço continua útil**. Inventar `DEGRADED` foi barato; o difícil foi decidir o que entra no `readiness`, porque essa lista é a definição operacional de "essencial".

A segunda lição veio da verificação, e é sobre confiança. Os dois indicadores foram escritos no mesmo dia, com a mesma estrutura, e **um funciona e o outro não**. O de circuitos reporta o estado real — provado abrindo um circuito e vendo o endpoint virar `DEGRADED`. O de cache reporta `UP` **sem abrir conexão com o Redis** — provado por `CLIENT LIST` e por contagem de comandos no servidor. Os dois parecem igualmente corretos lendo o código. **Health check é a única categoria de código em que "parece certo" não vale nada: ele existe justamente para ser acreditado.**

E a terceira: `DEGRADED` devolve 200. O status foi criado, ordenado e reportado com esmero — e, para qualquer probe que olhe o código HTTP, ele não existe. **Boa parte do trabalho de observabilidade se perde no último centímetro**, entre o que o sistema sabe e o que ele consegue contar a quem pergunta.

> [`health-checks.md`](../04-infraestrutura/health-checks.md)

---

## Fase 18 — Teste de carga e o teto de threads (ago/2026)

Dezessete fases mediram **correção**. Esta é a primeira a medir **capacidade** — e a primeira em que a medição contrariou a expectativa três vezes seguidas.

| Marco | O que se aprende |
|---|---|
| k6 com cenários e executores | VU é **concorrência**, arrival rate é **taxa**. Só o segundo mede capacidade |
| `sleep()` em `constant-arrival-rate` | Não segura nada — só multiplica a demanda de VUs e faz o k6 descartar iterações em silêncio |
| `check()` × threshold | `check` só conta; **só threshold reprova**. O teste de compra passava com a API errando 100% |
| Thresholds por `{scenario:...}` | Sem o filtro, o smoke de 1 VU dilui o percentil do teste de carga |
| Perfil `volume` sem threshold | Num teste que existe para achar o ponto de quebra, um limite de latência não mede nada |
| **OOM antes do gargalo de thread** | Com 256M o container morria (exit 137) por volta de 1600 req/s |
| Lei de Little, medida | 10 threads ÷ 8,6ms ≈ **1156 req/s** — foi exatamente a vazão observada |
| **Threads virtuais pioraram 9×** | 1156 → 127 req/s, e o serviço travou de vez |
| `server.tomcat.threads.max` ignorado | Ligar threads virtuais faz a linha virar letra morta, sem aviso nenhum |

**A lição da fase:** o pool de threads era o único **controle de admissão** do sistema. Enquanto ele existia, o excesso de carga esperava *fora* da aplicação, na fila do TCP — e a saturação era bem-comportada: `p(95)` de 2,44s e **zero erros**. Threads virtuais removeram esse teto, milhares de requisições entraram de uma vez, e cada uma passou a segurar uma conexão do Hikari enquanto bloqueava num `@ConcurrencyLimit(10)` que não rejeita ninguém. A fila mudou de lugar — de barata para cara.

**Threads virtuais não deixam nada mais rápido. Elas deixam mais coisas esperarem ao mesmo tempo.** Se o que está adiante tem capacidade 10, deixar 5.000 requisições chegarem lá não é ganho, é dano.

A segunda lição é sobre o sistema, não sobre a tecnologia: havia **quatro limitadores de valor 10** — threads do Tomcat, pool do Hikari (default, nunca declarado), e os dois bulkheads da Fase 16 — configurados em três fases diferentes, por três motivos diferentes, sem que ninguém tivesse olhado os quatro juntos. Os bulkheads eram **código morto** enquanto o Tomcat tinha 10 threads: não existiam 11 threads para limitar. A primeira vez que dispararam, derrubaram o serviço.

A terceira fecha o ciclo com a fase anterior: o serviço travado continuava `Up` para o Docker, e o `/actuator/health` **também não respondia** — ele depende das mesmas threads. Um health check que compartilha o pool da aplicação não consegue reportar a exaustão desse pool. **Quando a resposta mais importa, ela não chega.**

> [`testes-de-carga-k6.md`](../03-testes-integracao/testes-de-carga-k6.md) · [`threads-e-concorrencia.md`](../04-infraestrutura/threads-e-concorrencia.md)

---

## Fase 19 — Imagens em S3 com URL pré-assinada (ago/2026)

O catálogo ganhou imagens. A decisão que organiza tudo é uma só, e é a resposta direta ao que a Fase 18 mediu: **nenhum byte passa pelo backend**.

| Marco | O que se aprende |
|---|---|
| Upload em duas fases | O serviço **autoriza**; o cliente envia direto ao S3. O `remoteFileName` é o que amarra as duas pontas |
| URL pré-assinada | Amarra método, bucket, chave, content-type e prazo — **e nada além disso** |
| `fileExists` antes de anexar | Como o arquivo não passou por aqui, o catálogo precisa **perguntar** se ele chegou |
| Nome do cliente descartado | O objeto vira um UUID: resolve colisão, *path traversal* e vazamento pelo nome |
| Porta em `application/`, adapters em `infrastructure/` | E `@ConditionalOnProperty` como **requisito de inicialização**, não estilo: duas `@Component` na mesma interface e o Spring não sobe |
| `mainImage` como invariante | A primeira imagem vira principal sozinha; remover a principal promove outra |
| URL de leitura derivada, não persistida | Endereço não é identidade — trocar de bucket vira configuração, não migração |
| LocalStack com `ready.d` | Bucket, CORS e massa carregados sozinhos no `docker compose up` |
| **O tamanho declarado não é imposto** | 391 KB entregues sob autorização de 100 bytes, aceitos sem reclamação |

**A lição da fase:** a arquitetura está declarada na assinatura da porta. `StorageProvider` não tem um único método que receba ou devolva bytes — e é isso, não um comentário, que impede alguém de "só passar o arquivo pelo controller" seis meses depois. **Quando a decisão importante cabe no tipo, ela para de depender de disciplina.**

A segunda lição é sobre o que se perde junto. Sem ver os bytes, o serviço **não pode** validar conteúdo: a checagem de tipo é por extensão do nome, e um executável renomeado para `.png` passa. Não é descuido — é o preço, e ele precisa estar escrito ao lado do benefício.

A terceira é a que só apareceu medindo: a URL assinada **carrega o nome do host** e vai para o navegador. `algashop-localstack` é nome de container, e a máquina de quem desenvolve não resolve — daí três linhas no arquivo `hosts`. É a consequência literal de tirar o backend do caminho: **o cliente precisa alcançar o mesmo endereço que o servidor usou para assinar.**

E uma quarta, de contabilidade: trazer o `mainImage` para a listagem custou o `$project` do pipeline. O documento inteiro voltou a trafegar do Mongo, e o `shortDescription` mudou de 50 para 15 caracteres **sem que nenhum contrato ou teste notasse** — o mesmo tipo de mudança silenciosa que já tinha feito o `score` sumir na Fase 15.

> [`armazenamento-de-arquivos.md`](../02-persistencia/armazenamento-de-arquivos.md)

---

## Fase 20 — Identidade, OAuth 2.1 e o Authorization Server (ago/2026)

Dezenove fases construíram um sistema que **não sabe quem está do outro lado**. Nasce o sexto repositório — e ele é quase só configuração: uma dependência, uma classe vazia e dois clientes em YAML.

| Marco | O que se aprende |
|---|---|
| Senha × certificado × token | Três credenciais, três respostas para "quem valida, quanto dura, e o que acontece se vazar" |
| Autenticar × autorizar | "Quem é você" e "o que você pode" são perguntas diferentes, e terceirizar a segunda quase nunca é certo |
| Os quatro papéis do OAuth 2 | E que *client* não é "o app do usuário" — é **quem pede o token** |
| Emitir × verificar | Quem valida **nunca vê a senha**: o segredo mora num lugar só |
| `client_credentials` primeiro | Sem usuário no fluxo, é o grant que ensina o vocabulário inteiro sem o ruído do redirecionamento |
| Escopo é **teto**, não papel | Limita o que o token pode; não substitui a regra de negócio, que decide depois |
| **Opaco × JWT** | Opaco é referência (valida por introspecção, revoga na hora); JWT é auto-contido (valida local, **não revoga**) |
| TTL de 5m no JWT | Não é arbitrário: sem revogação, **o tempo de vida é a janela de exposição** |
| `/oauth2/jwks` e o `kid` | Publicar a chave pública é o que torna a rotação possível sem deploy coordenado |
| OAuth 2.1 | As mudanças quase todas **retiram** opções — `implicit` e `password` foram removidos |

**A lição da fase:** a mesma propriedade aparece três vezes com nomes diferentes. No certificado, no JWT e no desenho todo do OAuth, **quem valida nunca precisa do segredo**. É isso que permite ter cinco serviços verificando credenciais sem que nenhum deles guarde uma — e é a razão de emitir e verificar serem papéis separados, não uma escolha de arquitetura entre outras.

A segunda lição é o formato do token, que parece detalhe e decide o resto. JWT compra independência (o resource server valida sozinho, e continua funcionando com o emissor fora do ar) e **paga em revogação** — um token vazado vale até expirar. Opaco compra controle e paga uma ida à rede por requisição. Os dois clientes registrados existem para tornar essa troca visível, e os TTLs de 15m e 5m são a consequência aritmética dela.

E a terceira, que precisa ser dita com clareza: **isto ainda não protege nada.** Emitir token não protege recurso enquanto ninguém exigir o token, e nenhum serviço tem uma linha de configuração de resource server. A ordem está certa — não dá para verificar o que não existe —, mas o ciclo está aberto.

O `product-catalog` já ganhou a **dependência** de resource server, sem configuração, e isso bastou para derrubar 4 testes com `401`. Vale como quarta lição: **o starter de segurança tranca a aplicação inteira só por estar no classpath.** É o único starter que faz algo drástico sem ser configurado — e é proposital, porque falhar fechado é a escolha certa quando o assunto é segurança.

> [`fundamentos-identidade-oauth2.md`](../05-seguranca/fundamentos-identidade-oauth2.md) · [`authorization-server.md`](../05-seguranca/authorization-server.md)

---

## Fase 21 — Resource servers, escopos e a matriz de autorização (ago/2026)

A fase anterior terminou com "isto ainda não protege nada". Esta constrói **quem verifica** — e mostra que fechar metade de um ciclo pode ser pior que não ter começado.

| Marco | O que se aprende |
|---|---|
| `issuer-uri` nos três serviços | Uma linha configura tudo: dela o Spring descobre o `jwks_uri` pelo `.well-known` |
| O issuer fixado dos **dois** lados | Ele viaja no claim `iss` e quem valida **compara** — é o que barra token de outro emissor |
| `@EnableMethodSecurity` | Sem ele as anotações ficam decorativas: compila, sobe, e **toda rota fica aberta** |
| CSRF desligado — e por que está certo | O ataque depende de credencial que o **navegador envia sozinho**; um header `Authorization` não é |
| `permitAll` de caminho literal | `/actuator/health` não cobre `/readiness` — o probe levava **401** e a instância nunca entrava em rotação |
| **`@Valid` roda antes do `@PreAuthorize`** | Corpo inválido responde **400 antes de 403**: a autorização é a última camada, não a primeira |
| `SCOPE_` e a meta-anotação | O prefixo vem do `JwtGrantedAuthoritiesConverter`; e um typo na string **nega para sempre**, em silêncio |
| `products:stock:write` separado | Quem integra estoque não ganha de brinde o direito de reescrever preço |
| A matriz como teste | 43 rotas × 3 casos = **142 testes**, contra 2 que existiam antes |
| **O `ordering` chama o catálogo sem token** | E o 401 vira `Optional.empty()` → `ProductNotFound` → **422** |

**A lição da fase** é sobre ordem, e é desconfortável: **um ciclo de segurança fechado pela metade é pior que um ciclo aberto**. Antes, o catálogo não exigia token e tudo funcionava, inseguro e honesto. Agora ele exige, o `ordering` não manda, e o erro resultante — 422, "produto não encontrado" — aponta para o lugar errado. Segurança introduzida sem a ponta correspondente não deixa o sistema meio protegido: deixa o sistema quebrado com um sintoma enganoso.

A segunda lição é sobre **onde a decisão de autorização mora**. Ela parece vir primeiro e vem por último: filtro (401) → validação do corpo (400) → `@PreAuthorize` (403). Descobrir isso não foi leitura de documentação — foi a matriz falhando em 6 de 18 casos e obrigando a entender por quê.

E a terceira é o velho tema deste caderno em roupa nova: `hasAuthority('SCOPE_orders:raed')` **compila**. A segurança de um sistema inteiro apoiada numa string sem verificação de tipo é o mesmo problema do nome de cache da Fase 15 e do `project(Class)` errado da Fase 12 — **o que não é verificado em compilação precisa ser verificado por teste, ou não é verificado.**

> [`resource-server-e-escopos.md`](../05-seguranca/resource-server-e-escopos.md)

---

## Fase 22 — O `ordering` como OAuth2 client (ago/2026)

A fase anterior deixou o ciclo fechado pela metade, e um sintoma enganoso: 422 "produto não encontrado" onde a causa era 401. Esta fecha a outra ponta — e desfaz um acoplamento que tinha entrado junto.

| Marco | O que se aprende |
|---|---|
| Papel duplo | Ser resource server **não** torna um serviço capaz de chamar outro serviço protegido — são configurações independentes |
| As três peças | O `registration` diz **como**, o manager **executa**, o interceptor **anexa**. Sem a terceira, tudo parece configurado e nenhuma chamada leva header |
| `AuthorizedClientService...` × `Default...` | O segundo espera uma requisição HTTP em curso; máquina-para-máquina precisa funcionar em job e pool assíncrono |
| **Principal sintético** | O cache indexa por `(registrationId, principal)`; o principal padrão é o **usuário da requisição**, e cachear token de máquina por usuário fragmenta o cache |
| **`token-uri` × `issuer-uri`** | `issuer-uri` no lado *client* faz descoberta na **subida do contexto** — e acopla a inicialização ao authorization server |
| Uma fonte para o endereço | Os dois papéis interpolam do mesmo valor: divergir faz o serviço recusar o token que ele mesmo pediu |
| `catch` estreitado para `NotFound` | 401 e 403 deixaram de virar "produto não encontrado": agora são **502** |
| `unless = "#result == null"` | O Spring desembrulha o `Optional` antes do cache — o `unless` vê o valor, não o container |

**A lição da fase** é sobre acoplamento invisível: a configuração que parecia mais conveniente — `issuer-uri`, que descobre tudo sozinho — era a que amarrava a **subida** de um serviço à disponibilidade de outro. E o sintoma não dizia isso: dava um serviço que "não inicia", sem menção a segurança. **Conveniência de configuração e independência de deploy costumam estar em lados opostos**, e o lado certo depende do que se paga quando o outro serviço cai.

A segunda é sobre identidade em cache. O `principalResolver` só nomeia o dono da entrada de cache — misturá-lo com a autenticação da requisição faz um token **que não pertence a ninguém** ser guardado como se pertencesse a alguém, e o custo aparece como carga extra no authorization server, não como erro.

E a terceira já é um padrão, terceira vez seguida: **toda propriedade obrigatória acrescentada ao `development-env` quebra a suíte**, porque `src/test/resources` é uma árvore separada. Fase 19, Fase 21, Fase 22 — e o sintoma nunca aponta para a causa: o erro fala de um bean que falta, não de uma linha de YAML que ninguém copiou.

> [`oauth2-client-e-token.md`](../05-seguranca/oauth2-client-e-token.md)

---

## Fase 23 — Authorization code, consentimento e o estado que sobrou (ago/2026)

Todo o OAuth anterior foi **sem usuário**. Esta fase traz a pessoa para dentro do fluxo — e, com ela, a primeira coisa neste servidor que não pode sumir num restart.

| Marco | O que se aprende |
|---|---|
| `authorization_code` | `client_credentials` responde "este **serviço** pode"; só este responde "esta **pessoa** autorizou" |
| Por que existe um *código* no meio | O que volta no redirect viaja pela URL — histórico, `Referer`, log de proxy. Token ali estaria exposto; código sozinho não vale nada |
| O `state` | Vai e volta idêntico: é o que permite ao cliente recusar um retorno que ele não começou |
| Consentimento ≠ autenticação | Login pergunta *quem é você*; consentimento pergunta *o que eu deixo este app fazer por mim* |
| Escopo concedido × pedido | Pedi dois, consenti um, e o token saiu com **um** — o escopo é a interseção |
| Consentimento acumulativo | Uma linha por `(cliente, pessoa)`; escopo já consentido **não pergunta de novo** |
| **`reuse-refresh-token` no singular** | Propriedade ignorada em silêncio, default = reusar: **a rotação estava desligada** |
| O servidor ganhou banco | Não por escala — por consentimento: decisão de usuário que some no deploy nunca foi decisão |
| O schema vem da biblioteca | A migration não versiona a **nossa** modelagem; versiona a do Spring Authorization Server |

**A lição da fase** é a mais repetida deste caderno, com roupa nova: **a configuração afirmava uma propriedade de segurança que o sistema não tinha.** `reuse-refresh-token` (singular) é desconhecido para o Spring, propriedade desconhecida não gera erro, e o default é *reusar*. O fluxo funcionava, o comentário no YAML dizia "sempre revoga um refresh token antigo", e reusar um refresh rotacionado devolvia **200**. Uma letra depois — o plural —, o mesmo teste devolve `400 invalid_grant`.

É o mesmo mecanismo do nome de cache da Fase 15, do `project(Class)` da Fase 12 e do `hasAuthority('SCOPE_...')` da Fase 21. **O que não é verificado em compilação precisa ser verificado por comportamento** — e configuração *nunca* é verificada em compilação.

A segunda lição é sobre o que obriga um sistema a ter estado. Não foi volume nem performance: foi o fato de que **consentimento é uma decisão de uma pessoa**, e uma decisão que o deploy apaga não é uma decisão. Token em memória era aceitável enquanto ninguém precisava lembrar de nada.

E a terceira veio de um erro cometido ao documentar: **não se edita migration já aplicada.** Acrescentar um comentário no topo do `.sql` mudou o checksum e o Flyway recusou subir. O comentário virou documentação — que é onde ele deveria estar desde o começo.

> [`authorization-code-e-consentimento.md`](../05-seguranca/authorization-code-e-consentimento.md)

---

## Fase 24 — OpenID Connect: identidade, sessão e logout (ago/2026)

A fase anterior soube **o que a pessoa autorizou**. Esta sabe **quem ela é** — e descobre que identidade traz estado junto.

| Marco | O que se aprende |
|---|---|
| OIDC sobre OAuth2 | Access token responde *"pode fazer isto"*; ID token responde *"é esta pessoa"*. Públicos diferentes |
| O erro clássico | Mandar ID token para a API. Ele identifica, não autoriza |
| `openid` não vai ao consentimento | Consentimento é sobre **permissão em recurso**; pedir identidade não é |
| Usuário no banco | `auth_user`, `UserDetailsService`, e o fim do usuário em memória |
| `{noop}` e `{bcrypt}` juntos | O **prefixo** é o que permite migrar de algoritmo sem invalidar senha |
| Duas filter chains | A do protocolo antes da do `formLogin`; e o entry point escolhido por **negociação de conteúdo** |
| `sub` vira UUID | Identificador estável atravessa a fronteira — e é **mudança de contrato** |
| Logout revoga autorizações | O padrão encerra a **sessão**; revogar token é decisão da aplicação |
| **Logout não invalida JWT emitido** | `/userinfo` recusa, refresh recusa — e o resource server aceita até o `exp` |
| Sessão em banco | Sem ela, deploy desloga todo mundo e a segunda instância não reconhece ninguém |

**A lição da fase** é sobre onde a revogação chega: o logout apagou as autorizações, `/userinfo` passou a responder **401** e o refresh **400** — e o mesmo access token continuou sendo aceito por qualquer resource server, porque ele valida **localmente** pela assinatura e não pergunta nada a ninguém. **Os 5 minutos de TTL *são* a janela de logout incompleto.** É o preço do JWT, escolhido lá na Fase 20, chegando à conta.

A segunda é sobre um rename. `PersistenceConfig` virou `OAuth2PersistenceConfig`, mudou de pacote e **perdeu o `@Configuration`**. Sozinho, isso teria desfeito a Fase 23 em silêncio — tokens e consentimentos de volta à memória, sem erro, com o sintoma aparecendo semanas depois. O que salvou foi o código novo **injetar** `OAuth2AuthorizationService`: a dependência explícita transformou uma regressão invisível numa falha de inicialização. **Depender de um bean por injeção falha alto; depender dele por efeito colateral falha baixo.**

E a terceira, terceira vez seguida: `spring.session.timeout: 30m` está **inerte**, porque `@EnableJdbcHttpSession` faz a auto-configuração recuar. Os dois valores coincidem, então nada quebra — que é exatamente o que torna esse tipo de erro difícil de encontrar.

> [`openid-connect-e-sessao.md`](../05-seguranca/openid-connect-e-sessao.md)

---

## Fase 25 — Gestão de usuários, `/me` e auditoria com identidade real (ago/2026)

A Fase 24 pôs o UUID do usuário no `sub` do token. Esta **usa** esse `sub` — e ao usá-lo fecha a pendência mais antiga do caderno, aberta na Fase 8.

| Marco | O que se aprende |
|---|---|
| API de usuários no authorization server | O serviço de protocolo ganhou domínio próprio: listar, criar, atualizar, anonimizar |
| `SecurityCheckApplicationService` | A aplicação pergunta "quem está chamando?"; só a infraestrutura sabe que existe JWT |
| Auditoria com autor real | `created_by_user_id` = `sub` do token. **Dezessete fases depois** |
| `Optional.empty()` para máquina | "Criado por ninguém" é mais honesto que "criado por um UUID aleatório" |
| `/me` | O id que o **cliente não escolhe** elimina a classe de bug que `/{userId}` obriga a proteger |
| Máquina em `/me` → **403** | Não é falta de permissão, é ausência de sujeito |
| `@PreAuthorize("@securityCheck…")` | O SpEL resolve **bean por nome**, em runtime. Renomear quebra sem erro de compilação |
| `DELETE` anonimiza | A linha sobrevive porque o id dela pode estar gravado como autor em outros serviços |
| `aud` × `sub` como heurística | Distingue máquina de pessoa por **coincidência estrutural**, não por afirmação |

**A lição da fase** veio de três bugs encadeados que só um deles se via. `getAudience()` devolve `null` quando o token não traz `aud` — e como o `AuditorAware` consulta a segurança **antes de cada persistência**, o NPE derrubava todo `POST`/`PUT`/`DELETE` de todos os serviços. Atrás dele, um `catch (IllegalAccessError)` protegendo um `UUID.fromString` que lança `IllegalArgumentException`: código morto, escrito com a melhor das intenções, esperando o dia em que o bug da frente fosse corrigido. E os dois estavam nas **quatro** cópias idênticas do arquivo — a duplicação entre serviços cobrando quatro edições para um conserto.

A quarta, no authorization server, é a mesma família do rename da Fase 24: `TestSecurityConfig` declarava exatamente o bean que faltava, estava versionado, compilava — e **ninguém o importava**. `@TestConfiguration` não entra por component scan.

E a terceira asserção que envelheceu: `assertThat(createdByUserId).isNotNull()` passava **sempre** enquanto o autor era `UUID.randomUUID()`. Nunca teve chance de falhar, e por isso nunca provou nada. No `ordering` ela foi **fortalecida** para `isEqualTo(TEST_USER_ID)` — com `isNotNull()` o teste passaria igual se a auditoria regredisse ao UUID aleatório.

> [`gestao-de-usuarios-e-auditoria.md`](../05-seguranca/gestao-de-usuarios-e-auditoria.md)

---

## Fase 26 — PKCE, clientes públicos e silent refresh (ago/2026) ← etapa atual

Todos os clientes até aqui guardavam um segredo. Esta fase acrescenta o primeiro que **não pode guardar nada** — uma SPA — e descobre quanta coisa isso arrasta junto.

| Marco | O que se aprende |
|---|---|
| O problema do cliente público | Segredo embutido em bundle JavaScript está a um *view-source* de distância. Não é segredo, é string |
| PKCE | Em vez de segredo permanente, um **descartável por requisição**: `S256(verifier)` vai no `/authorize`, o verifier vai no `/token` |
| O que ele protege | **Continuidade**, não identidade: quem pediu o código é quem o troca. Não substitui redirect URI registrada |
| `plain` anula o mecanismo | Se o challenge é o próprio verifier, interceptar o `/authorize` entrega tudo |
| Sem `code_verifier` → **`invalid_client`** | Não `invalid_grant`. Para cliente público, o verifier **é** a autenticação |
| Sem `refresh_token` no admin | Credencial longa no navegador reproduz o problema do segredo — daí o silent refresh |
| `prompt=none` + iframe | Renova o token pela **sessão**, sem tela. O access token curto é projeção da sessão |
| O domínio comum não é cosmético | Cookie com `Domain=algashop.local` é o que faz `auth.` e `admin.` compartilharem sessão |
| same-**site** ≠ same-**origin** | O iframe é cross-origin (precisa de CORS) e same-site (o `SameSite=Lax` deixa passar) |
| CORS ≠ `frame-ancestors` | Uma deixa a SPA **chamar**; a outra deixa o AS ser **embutido**. Confundir é o erro clássico |

**A lição da fase** veio de tentar quebrar o PKCE, que é o único jeito de prová-lo. Verifier errado devolve `invalid_grant`, como esperado. Mas **omitir o verifier devolve `invalid_client`** — e esse erro diz, com precisão, o que o PKCE é: para um cliente sem segredo, o verifier ocupa exatamente o lugar que o `client_secret` ocuparia. O servidor não reclama de um parâmetro faltando; reclama de que o cliente não se identificou.

A segunda veio do `prompt=none`. Sem sessão, a spec do OIDC manda devolver `login_required` para a `redirect_uri`. O que acontece é um **302 para `/login`**: o `/oauth2/authorize` exige autenticação na filter chain, o `ExceptionTranslationFilter` intercepta, e o endpoint onde o `prompt=none` seria lido nunca é alcançado. Mesma lição de ordem de camadas da Fase 21 — **a camada de fora decide primeiro, e não conhece as regras da de dentro** — com o agravante de que, dentro de um iframe escondido, o sintoma é silêncio.

E a terceira, pela quarta vez: propriedade obrigatória nova no `development-env` derruba o perfil de teste (Fases 19, 21, 22, 26). Desta vez, porém, a mensagem apontou **exatamente** para a causa (`BindValidationException ... algashop.security.cors: must not be null`), em vez de aparecer quatro beans adiante como na fase anterior. `@ConfigurationProperties` + `@Validated` falha cedo e no lugar certo — e foi por isso que a correção declarou os valores no `test-env` em vez de dar defaults à classe: o default faria o servidor subir sem CSP em silêncio.

> [`pkce-e-clientes-publicos.md`](../05-seguranca/pkce-e-clientes-publicos.md)

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
- Replica set local e transação multi-coleção, coberta por teste de rollback
- Histórico de movimentação de estoque (`stock_movements`)
- Suíte de integração sobre Testcontainers — não depende mais de infraestrutura de pé
- Cache Redis server-side no catálogo e client-side no `ordering`, com cabeçalhos HTTP e `304`
- Timeout, retry, bulkhead, circuit breaker e fallback nas três integrações de saída
- Health check com Actuator, readiness por dependência essencial e status `DEGRADED`
- Ambiente de teste de carga com k6 — smoke, load e volume, com o compose subindo os quatro serviços
- Imagens de produto em S3 (LocalStack local), com upload direto por URL pré-assinada
- Authorization server emitindo token JWT por `client_credentials`, com 16 escopos granulares
- Os três serviços como resource servers, com escopo por rota e 142 testes travando a matriz
- O `ordering` como OAuth2 client: token por `client_credentials`, cacheado e anexado por interceptor
- Fluxo `authorization_code` com usuário, consentimento granular e refresh token com rotação, persistidos em Postgres
- OpenID Connect: ID token com claims de identidade, `/userinfo`, logout com revogação e sessão em banco
- API de usuários com filtros, paginação, `/me` e anonimização — e auditoria com o autor real, vindo do token
- Cliente público com PKCE, silent refresh por `prompt=none`, e os cinco serviços rodando no compose

**Próximos passos naturais:**
- **Entregar a senha temporária** — hoje ela vai para o stdout por `System.out.println` e não chega a ninguém; o usuário criado pela API não consegue logar
- **Ligar o PKCE também no client confidencial** — o admin já usa; o `algashop-ecommerce-web` segue com `require-proof-key: false`
- **Fazer `prompt=none` responder `login_required`** — hoje ele redireciona para `/login` e o iframe da SPA fica em silêncio
- **Claim explícito de tipo de token** — hoje "máquina ou pessoa?" é deduzido comparando `aud` e `sub`
- **Pôr o authorization server no compose** — no perfil `docker` o `ordering` não alcança o issuer
- **Validar audiência (`aud`)** — hoje um token vale em qualquer um dos três serviços
- **Verificar a origem do webhook do FastPay** — ele muda estado de fatura sem autenticação nenhuma
- **Persistir a chave de assinatura** — hoje cada reinício invalida todo JWT emitido
- **`authorization_code` + PKCE e um usuário de verdade** — o fluxo com pessoa ainda não existe
- **Testes para imagens e storage** — hoje são zero, e o `StorageProviderFakeImpl` existe exatamente para isso sem ser usado por nenhum
- **Recolher objetos órfãos no bucket** — entre autorizar e reivindicar, o arquivo pode ficar sem dono
- **Tirar as duas chamadas HTTP de dentro da `@Transactional` do `buyNow`** — o `billing` já fez o equivalente na Fase 16
- **Declarar o pool do Hikari** e redimensionar os `@ConcurrencyLimit` a partir de medição, não do default
- **Teste automatizado do cache** — hoje nenhum `*IT` exercita `@Cacheable`/`@CacheEvict`, e a corretude depende de leitura de código
- Métrica de taxa de acerto do cache (`keyspace_hits`/`keyspace_misses` existem e ninguém coleta)
- Cache nos perfis `docker` e `production`, que hoje rodam sem nenhum
- Mensageria real entre serviços (hoje os eventos são internos ao processo, aqui e no `ordering`)
- Retentativa, dead letter e reconciliação para a propagação da categoria e para os eventos de estoque
- Outbox para o evento que precisar sair do serviço — a transação resolve as escritas locais, não a integração
- Endpoint de histórico de movimentação (a coleção existe e ninguém a lê)
- Contratos Spring Cloud Contract para `/restock` e `/withdraw`
- Perfis `docker-env` e `production-env`, hoje referenciados e inexistentes
- Reordenar os estágios do pipeline: paginar **antes** do `$project`
- Índices na coleção `categories` (hoje a busca por nome é varredura com regex)
- Devolver `brand` à busca por termo (`@TextIndexed`) e tirar o índice que ficou órfão
- Testes do pipeline e um que afirme `IXSCAN` no `explain` (o agregado `Product` já tem o seu)
- ~~Autenticação~~ — começou na Fase 20, chegou aos resource servers na 21 e à auditoria na 25: o `UUID` aleatório deu lugar ao `sub` do token
- Observabilidade — tracing distribuído, métricas (a começar por `hikaricp.connections.pending` e `tomcat.threads.busy`), log estruturado

---

## O padrão que se repete

Olhando as vinte e seis fases em conjunto, a ordem foi sempre a mesma:

```
domínio → testes → persistência → API → contrato → infraestrutura → refatoração
```

Nunca o contrário. A tecnologia entrou depois que o problema estava entendido, e cada refatoração grande só aconteceu com rede de testes por baixo.

A Fase 14 é a primeira em que a seta chegou até o fim da linha: o requisito de domínio não parou na persistência, foi até o `docker-compose`. E foi lá que a segunda metade da frase cobrou o preço — a rede de testes existia, mas a mudança de infraestrutura a desligou sem avisar. **Ter teste e rodar teste são coisas diferentes**, e a diferença só aparece quando o chão se move.
