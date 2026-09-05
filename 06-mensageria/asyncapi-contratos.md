# AsyncAPI: o contrato da mensageria, escrito

> Três fases de Kafka deixaram um incômodo registrado em toda pendência: o contrato dos eventos era **conhecimento implícito** — espalhado entre POJOs copiados à mão, linhas de `type.mapping` em dois `application.yml` e a memória de quem escreveu. A Fase 40 escreve esse contrato: `docs/asyncapi/` descreve os eventos do tópico `product-catalog.product.events` em **AsyncAPI 3.1.0** — payload, headers, key, tópico, partições — em documentos que uma ferramenta lê, valida e renderiza. É a primeira metade da resposta à pendência de contrato; a segunda (um teste que impeça o documento de mentir) segue aberta — e este doc mostra por que ela importa: o primeiro yml já nasceu mentindo.
> Código real: `docs/asyncapi/product-catalog.yml` (produtor), `docs/asyncapi/ordering.yml` (consumidor), `docs/asyncapi/messages.yml` (as mensagens, compartilhadas).

---

## Para que serve — o OpenAPI da mensageria

O OpenAPI descreve APIs de **requisição/resposta**: rotas, verbos, payloads. Ele não tem vocabulário para o mundo assíncrono — não existe "endpoint" num tópico Kafka, não existe "resposta" para um evento publicado. O **AsyncAPI** é a especificação irmã para esse mundo: descreve **canais** (tópicos, filas), **operações** (publicar, consumir), **mensagens** (payload + headers + key) e **servidores** (o broker), num YAML machine-readable.

O que isso compra, concretamente:

- **Uma fonte de verdade legível.** Antes: para saber o que trafega no tópico, era preciso abrir 4 POJOs em dois repositórios, dois `type.mapping` e o `KafkaConfig`. Agora: um documento por serviço diz o que ele publica/consome, com que forma, e com que header.
- **Documentação renderizada de graça.** O mesmo YAML vira uma página navegável no Studio ou um HTML estático via CLI — ninguém mantém a documentação "bonita" à mão.
- **Validação estrutural.** `asyncapi validate` pega `$ref` quebrado, campo fora do schema, binding malformado — o contrato tem, ele próprio, um contrato.
- **A porta para contract testing.** Um documento machine-readable é o pré-requisito para um dia um teste comparar o que o código faz com o que o contrato promete. Sem o documento, não há o que testar.

O que ele **não** faz sozinho — e é o limite honesto desta fase: **nada liga o YAML ao código**. O `JacksonJsonSerializer` não lê o AsyncAPI; o listener não valida contra ele. É contrato-como-documentação, não contrato-como-teste — o rigor do Spring Cloud Contract que o lado REST tem ([stubs e contract tests](../03-testes-integracao/stubs-contract-tests.md)) ainda não chegou aqui.

## A anatomia do documento — e a inversão do 3.x

```yaml
asyncapi: 3.1.0          # a versao da SPEC (nao do contrato)
info:                     # titulo e versao DO CONTRATO
servers:                  # o broker: host + protocol kafka
channels:                 # o SUBSTANTIVO: o topico, com endereco e mensagens
operations:               # o VERBO: send | receive, apontando para o canal
components:               # o reuso: messages, schemas, messageTraits
```

A separação `channels` × `operations` é a grande mudança do AsyncAPI 3: o **canal é o substantivo** (o tópico `product-catalog.product.events` existe, tem 3 partições, carrega estas mensagens) e a **operação é o verbo** que cada serviço conjuga sobre ele — `action: send` no contrato do catálogo, `action: receive` no do ordering. O mesmo canal, descrito duas vezes, **uma por perspectiva**: é exatamente assim que a realidade é, porque produtor e consumidor não veem o tópico do mesmo jeito (o catálogo publica 5 tipos de mensagem; o ordering declara consumir 3 — o `ProductAdded` fica de fora **de propósito**, coerente com o handler default que o ignora).

## As decisões do projeto, mapeadas no YAML

| No YAML | O que documenta | De onde veio |
|---|---|---|
| `bindings.kafka: topic/partitions: 3/replicas: 3/min.insync.replicas: 2` | A topologia do tópico | O `NewTopic` do `KafkaConfig` e o cluster da [Fase 38](./kafka-na-pratica.md) |
| `messageTraits.KafkaProductKey` (key = product id) | A key do record — a cronologia por agregado | O `getAggregateId()` do `IntegrationEvent` |
| `headers.__TypeId__` com `const` | O nome LÓGICO que resolve a desserialização | O `type.mapping` dos dois lados |
| `payload.required` + tipos | A forma do JSON | Os `@NotNull` do V2 ([Fase 39](./ecst-e-validacao-de-eventos.md)) |
| `messages.yml` compartilhado via `$ref: "./messages.yml#/..."` | A mensagem é UMA, vista por dois | O que os POJOs copiados nunca conseguiram: **arquivo único** |

Duas dessas linhas merecem destaque:

- **O `__TypeId__` virou contrato escrito.** Nas Fases 38-39 o nome lógico era o fio invisível entre os serviços — duas strings iguais em dois YAMLs de configuração, sem nada que as declarasse como contrato. Agora há um `const` num documento cujo propósito é ser lido: quem for escrever um consumidor novo descobre o nome no contrato, não caçando no repositório do produtor.
- **O `messages.yml` compartilhado é a primeira coisa que os DOIS lados leem.** Os POJOs são cópias (decisão antiga: sem jar compartilhado); o `type.mapping` são duas listas paralelas. O arquivo de mensagens é o primeiro artefato **único** do contrato — o `$ref` relativo faz produtor e consumidor apontarem para a mesma definição.
- E é por isso que os contratos moram em **`docs/asyncapi/`**, não dentro de um serviço: o contrato pertence à fronteira, não a um dos lados dela.

## Evolução de contrato: o `deprecated` e o V2 lado a lado

O `messages.yml` descreve um `ProductPriceChangedEvent` marcado `deprecated: true` convivendo com o `ProductPriceChangedEventV2` no mesmo canal. **Honestidade didática: esse V1 é ilustrativo** — no código o evento de preço [nasceu V2](./ecst-e-validacao-de-eventos.md), e nenhum V1 jamais foi publicado. Ele está no contrato como exemplo de exatamente como um contrato real envelheceria: a mensagem antiga não some do documento enquanto puder haver eventos dela retidos no log (o consumidor novo que reler o tópico precisa saber o que vai encontrar); ela ganha `deprecated: true`, o V2 entra ao lado, e os dois constam do canal — a mesma convivência que o `__TypeId__` versionado permite no tópico, agora visível na documentação. Por ser fictício, o V1 aparece só no contrato do produtor; o do ordering declara apenas o que ele consome de verdade.

## Como usar — as três portas de entrada

**1. AsyncAPI Studio (zero instalação)** — <https://studio.asyncapi.org>: cole o conteúdo do YAML (ou abra o arquivo) e o painel direito renderiza a documentação navegável — operações, mensagens com payload expandido, servidores. Atenção ao detalhe local: o Studio no navegador não resolve `$ref` para arquivo relativo (`./messages.yml`) — para visualizar os contratos deste projeto lá, é preciso colar o conteúdo de `messages.yml` inline ou usar a CLI, que resolve refs de disco.

**2. AsyncAPI CLI (validação e geração)** — sem instalar nada permanente:

```bash
# valida o contrato e tudo que ele referencia ($refs inclusos)
npx -y @asyncapi/cli validate docs/asyncapi/product-catalog.yml
npx -y @asyncapi/cli validate docs/asyncapi/ordering.yml

# gera documentação HTML estática a partir do contrato
npx -y @asyncapi/cli generate fromTemplate docs/asyncapi/product-catalog.yml \
    @asyncapi/html-template@3.0.0 --use-new-generator -o build/asyncapi-docs
```

Os dois contratos deste repositório **passam no `validate`** — foi o critério de pronto desta fase.

**3. Extensão do editor** — a extensão *asyncapi-preview* (VS Code) e o suporte do IntelliJ dão preview renderizado e autocomplete de schema enquanto se edita o YAML.

## Armadilhas — o contrato que mente

O primeiro rascunho destes contratos entregou, de bandeja, a lição inteira sobre o limite de contrato-como-documentação:

- **O `__TypeId__` do V2 saiu errado** (`ProductPriceChangedEventV2...` em vez de `ProductPriceChangedV2...`). O documento mentia sobre **exatamente o campo que resolve a desserialização** — quem implementasse um consumidor pelo contrato cairia no handler default sem entender por quê. E nada quebrou ao escrever errado, porque **nada valida o YAML contra o código**: drift silencioso é o modo de falha padrão de todo contrato que é só documentação. (O `validate` da CLI não pega isso — o YAML errado era estruturalmente perfeito.)
- **Mensagens reais fora do contrato.** `ProductListed`/`ProductDelisted` — publicadas desde a Fase 38 e consumidas com efeito de negócio desde a 39 — não estavam em nenhum dos dois documentos. Contrato parcial é pior que ausente: passa a falsa sensação de completude.
- **Copy/paste entre perspectivas.** O contrato do ordering saiu com `summary: "Publishes product events"` numa operação `receive` — e listando o V1 que ele não consome. Perspectiva é o conceito central do AsyncAPI 3; é também o que o copy/paste destrói primeiro.

Todas consertadas nesta fase — mas o padrão é claro: **escrever o contrato é fácil; mantê-lo verdadeiro exige um teste**, e ele está nas pendências.

## Pendências registradas

- [ ] **Travar o contrato no CI** — `asyncapi validate` no pipeline é a parte barata; a valiosa é um teste que compare o YAML com o código (consts do `__TypeId__` × `type.mapping`, `required` × campos dos POJOs) — o equivalente de mensageria do que o SCC faz no REST
- [ ] **Gerar o HTML no pipeline** — documentação renderizada publicada a cada mudança de contrato
- [ ] **Declarar o destino do ProductAdded** — nenhum contrato `receive` o lista (coerente com o handler default do ordering); quando ganhar consumidor, o contrato do serviço novo nasce junto
- [ ] **`correlationId` nas mensagens** — o campo existe na spec e a correlação ponta a ponta segue pendente desde os fundamentos de EDA
- [ ] **Avaliar schema registry/Avro** — o passo além do JSON schema inline, quando os contratos multiplicarem

## Checklist de revisão

- [ ] Sei explicar o que o AsyncAPI resolve que o OpenAPI não alcança — e o que ele NÃO resolve sozinho (nada valida contrato × código)
- [ ] Entendo a inversão do 3.x: canal = substantivo, operação = verbo — e por que cada serviço tem seu documento com sua perspectiva (`send` × `receive`)
- [ ] Sei onde cada decisão das Fases 38-39 aparece no YAML (key, `__TypeId__`, topologia do tópico, `required`)
- [ ] Sei por que o `messages.yml` compartilhado é diferente de tudo que o contrato tinha antes (arquivo único × cópias paralelas)
- [ ] Sei usar as três portas: Studio, `asyncapi validate`/`generate`, extensão do editor
- [ ] Consigo contar a história do contrato que mentia — e por que o `validate` não a teria evitado

## Referências

- [AsyncAPI Specification 3.1.0](https://www.asyncapi.com/docs/reference/specification/v3.1.0) (a spec — canais, operações, components, traits)
- [Kafka Bindings](https://github.com/asyncapi/bindings/tree/master/kafka) (topic, partitions, key — o vocabulário Kafka do contrato)
- [AsyncAPI Studio](https://studio.asyncapi.org) · [AsyncAPI CLI](https://www.asyncapi.com/tools/cli)
- [Kafka na prática](./kafka-na-pratica.md) · [ECST e validação de eventos](./ecst-e-validacao-de-eventos.md) · [Stubs e contract tests (REST)](../03-testes-integracao/stubs-contract-tests.md) · [Fundamentos de EDA](./fundamentos-eda.md)
