# Índices no MongoDB

> Como o `product-catalog` deixou de varrer a coleção inteira a cada listagem: índice composto, índice parcial e índice de texto declarados no próprio agregado.
> Código real: `domain/product/Product.java`, `application.yml`, `ProductQueryServiceImpl.java`.

---

## O problema

O doc de [consultas dinâmicas](./consultas-mongo-criteria.md) terminou com uma pendência registrada em letras grandes:

> *"Nenhum índice foi criado para os campos filtrados (`enabled`, `salePrice`, `addedAt`, `categoryId`). Com a coleção pequena não aparece; com volume, toda consulta é varredura."*

Esta etapa é a resposta a ela.

O detalhe é que o problema **não dava para ver**. Com o `products.json` de carga — algumas dezenas de documentos — varrer tudo e usar índice levam o mesmo tempo. Foi por isso que entrou o `db/testdata/products-large.json`, com 560 mil produtos: sem massa, não há o que medir, e "otimização" vira palpite.

---

## Como o MongoDB decide

Toda consulta gera um **plano de execução**, e ele começa de um jeito ou de outro:

| Estágio | O que é | Custo |
|---|---|---|
| `COLLSCAN` | varredura de coleção — lê **todo** documento e testa um a um | proporcional ao tamanho da coleção |
| `IXSCAN` | percorre o índice e só busca os documentos que casaram | proporcional ao tamanho do **resultado** |

Quem conta a verdade é o `explain`:

```javascript
db.products.find({ categoryId: UUID("..."), enabled: true, salePrice: { $gte: 100 } })
           .explain("executionStats")
```

Os dois números que importam na saída:

- **`totalKeysExamined`** — quantas entradas de índice foram lidas;
- **`totalDocsExamined`** — quantos documentos completos foram lidos.

O ideal é `totalDocsExamined` próximo de `nReturned`. Se `totalDocsExamined` for o tamanho da coleção e `totalKeysExamined` for `0`, é `COLLSCAN` puro: o índice não existe, ou existe e não foi escolhido.

> Um índice existir não garante que ele será usado. A seção do [índice parcial](#índice-parcial-menor-mas-mais-exigente) é exatamente um caso em que ele existe e o Mongo, com razão, o ignora.

---

## Declarar índice por anotação

No Spring Data, o índice é declarado junto do campo que ele indexa — a definição fica ao lado da modelagem, não num script à parte:

| Anotação | Onde vai | Gera |
|---|---|---|
| `@Indexed` | campo | índice simples de um campo |
| `@CompoundIndex` | classe (repetível) | índice de vários campos, na ordem dada |
| `@TextIndexed` | campo | participação no índice de texto da coleção |
| `@TextScore` | campo | **não** é índice — recebe a relevância calculada na busca |

E a chave que faz tudo isso sair do papel:

```yaml
spring:
  mongodb:
    uri: mongodb://...          # a CONEXAO, que no Boot 4 mudou de prefixo
  data:
    mongodb:
      auto-index-creation: true # a criacao dos indices, que NAO mudou
```

Sem `auto-index-creation`, as anotações são decorativas: nenhum índice é criado, nenhum erro é levantado, e a aplicação sobe normalmente varrendo a coleção.

> ⚠️ **Pegadinha de versão.** No Spring Boot 4 a propriedade de conexão virou `spring.mongodb.uri` (era `spring.data.mongodb.uri`), mas `auto-index-creation` **continua** em `spring.data.mongodb`. Os dois blocos convivem no mesmo YAML, e errar o prefixo não produz erro — só silêncio.

> ⚠️ **Não é para produção.** A criação roda na inicialização e a aplicação fica travada enquanto o índice é construído. Numa coleção grande isso é minutos de indisponibilidade a cada deploy. Em produção o índice é criado por fora, em janela controlada, e a aplicação sobe só assumindo que ele existe.

---

## Índice composto e a regra ESR

O agregado declara dois:

```java
@CompoundIndex(name = "pidx_product_by_category_enabledTrue_salePrice",
        def = "{'categoryId': 1, 'enabled': 1, 'salePrice': 1}",
        partialFilter = "{'enabled': true}")
@CompoundIndex(name = "pidx_product_by_category_enabledTrue_addedAt",
        def = "{'categoryId': 1, 'enabled': 1, 'addedAt': -1}",
        partialFilter = "{'enabled': true}")
public class Product { ... }
```

**A ordem dos campos não é estética.** A regra de bolso é **ESR** — *Equality, Sort, Range*:

| Posição | Tipo de uso | No índice acima |
|---|---|---|
| 1º | **E**quality — igualdade exata (`is`, `$in`) | `categoryId`, `enabled` |
| 2º | **S**ort — ordenação | `addedAt` (no segundo índice) |
| 3º | **R**ange — faixa (`$gte`, `$lt`) | `salePrice` (no primeiro) |

O motivo: um índice é uma lista ordenada. Enquanto os campos anteriores forem igualdade, todas as entradas que interessam ficam **contíguas** — o Mongo salta direto para o bloco certo. Assim que entra uma faixa, o bloco deixa de ser contíguo para os campos seguintes, e o que vier depois não consegue mais ser usado para ordenar sem reordenação em memória.

### Por que **dois** índices e não um

Porque um índice só serve bem uma dessas pontas por consulta:

- filtrar por faixa de `salePrice` → precisa de `salePrice` logo depois das igualdades;
- ordenar por `addedAt` → precisa de `addedAt` logo depois das igualdades.

Os dois no mesmo índice competem pela mesma posição. Com um índice só, uma das consultas filtraria pelo índice e **ordenaria em memória** — e o Mongo aborta ordenação em memória acima de **32 MB** com o erro `Sort exceeded memory limit`. Em coleção pequena passa despercebido; é exatamente o tipo de coisa que só aparece em produção.

**O `-1` do `addedAt`** é do mais recente para o mais antigo. Não obriga a consulta a ser `DESC`: o Mongo percorre o índice nos dois sentidos, então esse mesmo índice atende `ASC` — que é o default do `ProductFilter`.

---

## Índice parcial: menor, mas mais exigente

```java
partialFilter = "{'enabled': true}"
```

Só entra no índice o documento que satisfaz o filtro. Produto desativado não ocupa espaço, o índice fica menor e cabe melhor em memória — e como o catálogo quase sempre lista só produto ativo, é desperdício indexar o resto.

**A contrapartida:** o Mongo só escolhe um índice parcial quando consegue **provar** que a consulta é subconjunto do filtro parcial. Na prática:

| Consulta | Usa o índice parcial? |
|---|---|
| `{ categoryId: X, enabled: true }` | ✅ sim |
| `{ categoryId: X, enabled: false }` | ❌ não — pede o que não está no índice |
| `{ categoryId: X }` (sem `enabled`) | ❌ **não** — pode haver inativo no resultado |

A última linha é a armadilha. `ProductFilter.enabled` é opcional; cliente que não manda `?enabled=true` cai em varredura mesmo com o índice ali, criado e íntegro. Não é bug — é o Mongo sendo correto: o índice parcial não tem como responder por documento que ele não indexou.

---

## Índice de texto

```java
@TextIndexed(weight = 1)
private String name;

@TextIndexed(weight = 1)
private String description;
```

```java
query.addCriteria(TextCriteria.forDefaultLanguage().matching(filter.getTerm()));
// { $text: { $search: "notebook gamer" } }
```

Três regras que não são óbvias:

1. **Só existe UM índice de texto por coleção.** Não é uma limitação do Spring — é do MongoDB. Todos os campos `@TextIndexed` entram no mesmo índice; não dá para ter um índice de texto para `name` e outro para `description`.
2. **`weight` pesa a relevância** de cada campo no cálculo do score. Achar o termo num campo de peso 10 vale mais que achar num de peso 1. Peso só significa alguma coisa se os valores **diferirem** entre si.
3. **Casa palavra inteira, com stemming.** O Mongo quebra o texto em palavras, remove stop words e reduz ao radical no idioma configurado (`forDefaultLanguage()` = inglês). `"running"` acha `"run"`; `"note"` **não** acha `"notebook"`.

### O que mudou em relação ao regex

Esta busca era um `$or` de três regex. A troca foi um ganho grande com um custo real:

| | `$or` de regex (antes) | `$text` (agora) |
|---|---|---|
| Usa índice | não — varre a coleção | sim |
| Casamento | qualquer parte (`note` acha `notebook`) | palavra inteira, com stemming |
| Busca enquanto digita | funciona | **não funciona** |
| Ordenação por relevância | não existe | nativa (`score`) |
| Termo malicioso | ReDoS: `(a+)+$` avaliado no servidor | inofensivo |
| Campos cobertos | `name`, `brand`, `description` | `name`, `description` |

As duas últimas linhas resumem a troca: ganhou-se segurança e desempenho, perdeu-se a busca por marca e o autocomplete. Para catálogo com busca explícita ("buscar"), `$text` é o certo. Para caixa que filtra a cada tecla, regex ancorado (`/^note/`, que **usa** índice) ou um motor de busca dedicado seriam a resposta.

---

## Ordenar por relevância

```java
@TextScore
private Float score;
```

```java
if (StringUtils.isNotBlank(filter.getTerm())) {
    return Sort.by("score");   // vira { score: { $meta: "textScore" } }
}
```

O `score` é um campo estranho e vale entender por quê:

- **Não é persistido.** Não existe na coleção; nenhum `insert` grava isso.
- **É calculado por consulta.** O Mongo pontua cada documento pela relevância no `$text` daquela busca.
- **Fora de busca textual chega `null`.** Uma listagem comum devolve `score: null` — o campo entrou também no `ProductSummaryOutput`, então aparece no JSON da API sempre, preenchido só quando há termo.

A mágica está no `@TextScore`: é ele que faz o `QueryMapper` do Spring Data traduzir `Sort.by("score")` para `{ score: { $meta: "textScore" } }`. **Sem a anotação**, o Mongo receberia um pedido de ordenação por um campo que não existe em documento nenhum — sem erro, e sem efeito.

**A direção do `Sort` é irrelevante aqui.** Ordenação por `$meta: textScore` no MongoDB é sempre do mais relevante para o menos. `Sort.by("score")` e `Sort.by("score").descending()` produzem a mesma ordem.

---

## ⚠️ Índice e carga de dados

Esta linha do `DataLoader` mudou nesta etapa, e o motivo é inteiramente sobre índices:

```java
// antes
mongoOperations.getCollection(collectionName).drop();

// agora
mongoOperations.getCollection(collectionName).deleteMany(new BsonDocument());
```

A ordem dos eventos na subida é:

```
contexto Spring sobe
   └─ mapping context lê as anotações → cria os índices no Mongo
      └─ ApplicationRunner (DataLoader) roda → limpa e insere a massa
```

Os índices são criados **antes** do `DataLoader`. E `drop()` derruba a coleção **com os índices dentro** — a aplicação subiria, criaria os índices, e em seguida os destruiria por conta própria. Toda consulta seguinte viraria varredura, sem nenhum sintoma visível além da lentidão.

`deleteMany({})` remove os documentos e **preserva as definições de índice**. Mais lento que `drop()` numa coleção enorme (apaga documento a documento, respeitando o índice), e é o preço certo a pagar aqui.

Ver [`carga-de-dados-mongo.md`](../04-infraestrutura/carga-de-dados-mongo.md).

---

## Conferindo na prática

```javascript
// quais índices existem
db.products.getIndexes()

// a consulta usa índice?
db.products.find({ categoryId: UUID("..."), enabled: true }).explain("executionStats")

// quem está sendo realmente usado — 'accesses.ops' zerado = índice que só custa
db.products.aggregate([{ $indexStats: {} }])
```

Depois de subir a aplicação, o `getIndexes()` deve listar cinco: o `_id_` automático, o `idx_product_by_brand`, o índice de texto e os dois `pidx_*`.

O teste que fecha a lição é rodar o `explain` **duas vezes**: com `enabled: true` e sem. A primeira dá `IXSCAN`, a segunda `COLLSCAN` — a demonstração exata do que o `partialFilter` faz.

---

## Índice não é de graça

Índice é troca, não melhoria pura:

- **Escrita fica mais cara.** Todo `insert`/`update` precisa atualizar cada índice afetado. Cinco índices = cinco estruturas a manter por escrita.
- **Ocupa memória.** O ganho vem de o índice caber em RAM; índice que não cabe vira leitura de disco e devolve boa parte do benefício.
- **Índice não usado é só custo.** Paga escrita e memória sem acelerar nada — daí o `$indexStats` na seção anterior.

Por isso não se indexa "tudo por via das dúvidas". Indexa-se o que a consulta real usa, e confere-se com `explain` que foi mesmo usado.

---

## Armadilhas

1. **Anotação sem `auto-index-creation` não faz nada.** Nenhum erro, nenhum índice.
2. **Prefixo errado no Boot 4.** `spring.data.mongodb.auto-index-creation`, mesmo com a URI em `spring.mongodb`.
3. **Índice parcial exige o predicado explícito.** Consulta sem `enabled: true` não o usa.
4. **Ordem dos campos no composto importa.** ESR — igualdade, ordenação, faixa.
5. **`drop()` leva os índices junto.** `deleteMany({})` quando a intenção é só esvaziar.
6. **`$text` casa palavra inteira.** Quem esperava substring vai achar que a busca quebrou.
7. **`$expr` continua fora de qualquer índice.** O filtro `hasDiscount` não é indexável — ver [`consultas-mongo-criteria.md`](./consultas-mongo-criteria.md#comparar-dois-campos-do-mesmo-documento).
8. **Pesos iguais são pesos inexistentes.** `weight = 1` nos dois campos é o mesmo que não declarar peso.

---

## Pendências registradas

- [ ] **`brand` está indexado e ninguém usa.** O `@Indexed(name = "idx_product_by_brand")` foi criado quando a busca era regex sobre três campos. Com a troca para `$text`, nenhuma consulta filtra por marca — o índice só custa escrita e memória. Ou entra um filtro `?brand=`, ou o índice sai.
- [ ] **A busca por marca foi perdida.** `?term=Apple` achava produto pela marca no regex; com `$text` sobre `name`/`description`, não acha mais. A correção é uma linha — `@TextIndexed` em `brand` — já que o índice de texto é único e comporta o terceiro campo.
- [ ] **Os dois pesos do índice de texto são `1`.** Achar no nome deveria valer mais que achar na descrição; hoje valem igual. `weight = 10` em `name` resolveria.
- [ ] **A coleção `categories` não tem índice nenhum**, e o `CategoryQueryServiceImpl` filtra por `Criteria.where("name").regex(...)` sem âncora — varredura garantida. Hoje são poucas categorias; o padrão é o oposto do que o `Product` acabou de adotar. O `Pattern.quote` também continua faltando ali.
- [ ] **`auto-index-creation: true` está no `application.yml` default**, não num perfil de desenvolvimento — mesma situação do `data-load.auto-drop`.
- [ ] **O `products-large.json` não está referenciado** no `sources` do `data-load` (a entrada está comentada). Quem clonar o projeto e rodar o `explain` vai medir sobre dezenas de documentos e não ver diferença nenhuma.
- [ ] **Nenhum teste verifica o plano de execução.** Um teste de integração que rode `explain` e afirme `IXSCAN` seria a única forma de perceber que um índice parou de ser usado — hoje isso passaria em silêncio.

---

## Checklist de revisão

- [ ] Sei ler um `explain` e distinguir `COLLSCAN` de `IXSCAN`
- [ ] Sei o que significam `totalKeysExamined` e `totalDocsExamined`
- [ ] Sei a regra ESR e por que a ordem dos campos no índice composto importa
- [ ] Sei explicar por que são **dois** índices compostos e não um
- [ ] Sei em que situação um índice parcial deixa de ser usado
- [ ] Sei que só existe um índice de texto por coleção
- [ ] Sei por que `Sort.by("score")` funciona, e o que o `@TextScore` tem a ver com isso
- [ ] Sei por que `drop()` foi trocado por `deleteMany({})`
- [ ] Sei o que se paga por cada índice criado

---

## Referências

- [MongoDB — Indexes](https://www.mongodb.com/docs/manual/indexes/)
- [MongoDB — a regra ESR](https://www.mongodb.com/docs/manual/tutorial/equality-sort-range-guideline/)
- [MongoDB — Partial Indexes](https://www.mongodb.com/docs/manual/core/index-partial/)
- [MongoDB — Text Indexes](https://www.mongodb.com/docs/manual/core/indexes/index-types/index-text/)
- [MongoDB — `explain` results](https://www.mongodb.com/docs/manual/reference/explain-results/)
- [Spring Data MongoDB — Index Creation](https://docs.spring.io/spring-data/mongodb/reference/mongodb/mapping/mapping-index-management.html)
- [`consultas-mongo-criteria.md`](./consultas-mongo-criteria.md) — as consultas que estes índices servem
- [`product-catalog-mongo.md`](./product-catalog-mongo.md) — o agregado onde os índices são declarados
- [`carga-de-dados-mongo.md`](../04-infraestrutura/carga-de-dados-mongo.md) — a massa de teste e o `deleteMany`
