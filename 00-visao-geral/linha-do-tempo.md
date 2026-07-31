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
| — | `@DocumentReference` para categoria | Embutir vs. referenciar |
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

## Fase 10 — Índices e busca textual (jul/2026) ← etapa atual

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

**Próximos passos naturais:**
- Mensageria real entre serviços (hoje os eventos são internos ao processo)
- Índices na coleção `categories` (hoje a busca por nome é varredura com regex)
- Devolver `brand` à busca por termo (`@TextIndexed`) e tirar o índice que ficou órfão
- Testes do agregado `Product`, do `queryWith` e um que afirme `IXSCAN` no `explain`
- Autenticação (a auditoria usa um `UUID` aleatório como placeholder)
- Observabilidade — tracing distribuído, métricas, log estruturado
- Resiliência — circuit breaker e retry nas chamadas entre serviços

---

## O padrão que se repete

Olhando as dez fases em conjunto, a ordem foi sempre a mesma:

```
domínio → testes → persistência → API → contrato → infraestrutura → refatoração
```

Nunca o contrário. A tecnologia entrou depois que o problema estava entendido, e cada refatoração grande só aconteceu com rede de testes por baixo.
