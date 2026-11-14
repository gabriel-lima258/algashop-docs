# Replicando os clientes (desafio)

> **Em uma frase:** os clientes do ordering ganham o mesmo caminho dos pedidos: um source (`cdc.source.ordering.customer`), um sink (`cdc.sink.insights.customer`) e uma tabela `customer` no insights. A decisão central é **quais colunas** vão para o relatório. Ela nasce das perguntas do negócio e tem que ser igual em dois lugares: na lista do conector e na migration.

⬅️ Anterior: [Deletes, tombstones e tipos de dados](./14-deletes-tombstones-e-tipos-de-dados.md) · ➡️ Próximo: [Replicando os itens dos pedidos](./16-replicando-os-itens-dos-pedidos.md) · 🧭 Trilha: [README](../README.md#trilha-de-leitura)

---

## O problema

O pedido no insights tem `customer_id`, mas o relatório estratégico quer saber **quem** são esses clientes: de onde vêm, que idade têm, se aceitam promoções. Esse dado está na tabela `customer` do ordering, que também guarda nome, documento, e-mail, telefone e endereço completo, coisas que um relatório não deveria ter.

## A ideia: as colunas vêm das perguntas, não da tabela

| Pergunta do relatório | Coluna |
|---|---|
| Quantos clientes por cidade e por estado? | `address_city`, `address_state` |
| Como se distribuem por faixa de idade? | `birth_date` |
| Quantos pontos de fidelidade acumulam? | `loyalty_points` |
| Quantos aceitam promoções? | `promotion_notifications_allowed` |
| Quantos ativos, quantos arquivados? | `archived`, `archived_at` (ativo = não arquivado) |
| Quantos se cadastraram por mês? | `registered_at` |
| Ligar o pedido ao cliente | `id` (o mesmo id do ordering; o `customer_id` do pedido aponta para ele) |

**Ficam de fora:**

| O quê | Por quê |
|---|---|
| `first_name`, `last_name`, `document`, `email`, `phone` | dado de contato e de identificação: nenhuma pergunta precisa dele, e dado pessoal que não sai do serviço não vaza |
| `address_street`, `number`, `complement`, `neighborhood`, `zip_code` | do endereço, o relatório só usa cidade e estado |
| `created_by_user_id`, `last_modified_*`, `version` | controle e auditoria: não respondem a nenhuma pergunta |

São 9 colunas, de 23.

## Como funciona no AlgaShop

### A mesma lista em dois lugares

```
column.include.list (source)  ==  colunas da V002 (insights)
```

- **No source:** `"column.include.list": "public\\.customer\\.(id|birth_date|registered_at|loyalty_points|promotion_notifications_allowed|address_city|address_state|archived|archived_at)"`.
- **No insights:** a `V002__create-table-customer.sql` cria exatamente essas 9 colunas, com os mesmos nomes e os mesmos tipos da origem.

Se as duas listas se separarem:

| Divergência | O que acontece |
|---|---|
| Campo na mensagem **sem** coluna na tabela | o sink tenta gravar uma coluna que não existe; com `schema.evolution: none`, a task vai a `FAILED` |
| Coluna na tabela **sem** campo na mensagem | não dá erro: a coluna só fica nula em todas as linhas, e o relatório erra em silêncio |

O segundo caso é o pior, porque nada avisa. Vale conferir: nenhuma coluna da V002 pode estar nula em **todos** os clientes.

### Os tipos novos

| Coluna | Na mensagem | No insights |
|---|---|---|
| `id` | `string` + `io.debezium.data.Uuid` | `uuid` |
| `birth_date` | `io.debezium.time.Date` (número de dias desde 1970) | `date` |
| `registered_at`, `archived_at` | `io.debezium.time.ZonedTimestamp` | `timestamptz` |
| `archived`, `promotion_notifications_allowed` | `boolean` | `boolean` |
| `loyalty_points` | `int32` | `integer` |
| `address_city`, `address_state` | `string` | `varchar` |

### Dois conectores do Postgres com o mesmo `topic.prefix`

O conector de clientes lê o mesmo banco e usa o mesmo prefixo do de pedidos (`cdc.ordering`). O tópico novo fica `cdc.ordering.public.customer`. Para os dois conviverem, **nada** pode ser compartilhado:

| O quê | Pedidos | Clientes | Se repetir |
|---|---|---|---|
| Nome do conector | `cdc.source.ordering.order` | `cdc.source.ordering.customer` | o PUT de um sobrescreve o outro |
| Slot de replicação | `cdc_ordering_order` | `cdc_ordering_customer` | um slot só aceita um leitor de cada vez |
| Publication | `cdc_ordering_order` | `cdc_ordering_customer` | com `filtered`, um passaria a mexer na lista de tabelas do outro |
| Métricas (`custom.metric.tags`) | `table=order` | `table=customer` | o conector não sobe as métricas e **não faz o snapshot** |

**A armadilha das métricas.** Cada conector do Postgres registra métricas no JMX (andamento do snapshot, atraso na leitura do WAL), com um nome montado a partir do `topic.prefix`. Com o mesmo prefixo e sem nada que os diferencie, o segundo conector tenta registrar um nome que já existe. O sintoma engana:
- o status mostra conector e task `RUNNING`, sem erro;
- o snapshot nunca começa, e o tópico fica vazio;
- o único sinal é a linha `Unable to register metrics as an old set with the same name ... exists, retrying`, repetida no log do Connect.

O `custom.metric.tags` acrescenta uma etiqueta (`chave=valor`) ao nome das métricas, e assim cada conector ganha um nome único. O de pedidos já tinha `table=order` desde o [doc 10](./10-extraindo-pedidos-com-debezium.md); sozinho, ele não fazia diferença. Outra saída seria um `topic.prefix` diferente por conector, mas isso muda o nome dos tópicos.

### A ordem certa

1. Subir o insights, para o Flyway aplicar a V002. **A tabela precisa existir antes do sink.**
2. Rodar o init (`docker compose -f docker-compose.tools.yml up algashop-kafka-connect-init`). O script aplica os sources antes dos sinks, seja qual for a ordem dos arquivos ([doc 13](./13-automatizando-a-criacao-dos-conectores.md)).

### O sink de pedidos ganhou uma DLQ

Junto com o desafio, o sink de pedidos ganhou três linhas da aula de falhas nos conectores:

```json
"errors.tolerance": "all",
"errors.deadletterqueue.topic.name": "dlt.cdc.sink.insights.order",
"errors.deadletterqueue.context.headers.enable": "true"
```

- `errors.tolerance: all`: uma mensagem que falha (por exemplo, na conversão) **não derruba** a task.
- A mensagem com problema vai para o tópico `dlt.cdc.sink.insights.order`. É a mesma ideia da DLT da [mensageria](../06-mensageria/07-resiliencia-no-consumer.md).
- `context.headers`: a mensagem na DLT leva nos headers o motivo do erro, o tópico, a partição e o offset de origem.

O preço: com `all`, os erros não param o conector. Por isso alguém precisa **olhar a DLT**, ou o relatório perde linhas em silêncio. O sink de clientes ficou sem essa configuração.

## Como aplicar

1. Liste as perguntas do relatório e só então escolha as colunas.
2. Corte tudo o que é contato, identificação e controle.
3. Escreva a mesma lista no `column.include.list` e na migration, com os mesmos nomes e tipos.
4. Segundo conector no mesmo banco: slot, publication, nome e `custom.metric.tags` próprios.
5. Migration aplicada antes do init.
6. Confira: contagens iguais e nenhuma coluna nula em todas as linhas.

## Para lembrar

- A pergunta escolhe a coluna; a tabela de origem não.
- A mesma lista em dois lugares: no conector e na migration.
- Campo sem coluna → `FAILED`; coluna sem campo → nula em silêncio.
- Dois conectores, o mesmo prefixo: `custom.metric.tags` diferente, senão o snapshot não começa.
- Slot e publication nunca se dividem entre conectores.

## Cuidados

- **`RUNNING` não quer dizer que funciona.** No conflito de métricas, o status diz que está tudo bem. Confira o tópico e o log.
- **Colunas novas pedem duas mudanças.** Mudar a lista do conector sem uma V003 (ou o contrário) quebra o sink ou deixa colunas vazias.
- **Os arquivos entraram vazios no stage.** Os dois JSON e a V002 foram adicionados antes do conteúdo; foi preciso dar `git add` de novo.
- **A DLQ precisa de alguém olhando.** Com `errors.tolerance: all`, o erro não aparece no status.

## Pendências

- O teste com o ecommerce (criar uma conta, mudar o endereço e o nome) e o teste da remoção pelo restart do ordering: ficam para você rodar, veja abaixo.
- Os produtos (product-catalog, MongoDB).
- O primeiro relatório estratégico no insights cruzando `order` e `customer`.

## Testes

14/11/2026:

| Verificação | Resultado |
|---|---|
| V002 | aplicada pelo insights: `flyway_schema_history` com 001 e 002 |
| Log do init | `cdc.source.ordering.customer -> 201`, `cdc.sink.insights.customer -> 201`, os dois de pedidos `-> 200` |
| Status | os 4 conectores e as 4 tasks `RUNNING` |
| Log do Connect | nenhum `Unable to register metrics` |
| Slots | `cdc_ordering_customer` e `cdc_ordering_order`, os dois ativos |
| Snapshot | 22 clientes no ordering = 22 no insights |
| Colunas | nenhuma coluna da V002 nula em todos: `birth_date` 21 de 22 (um cliente já não tem data na origem), `archived_at` 1 (o único arquivado) |
| Mensagem do tópico | só os 9 campos; `birth_date` como `io.debezium.time.Date`; sem nome, documento, e-mail, telefone ou rua |
| Mudar cidade e estado (no banco, cliente `6e148bd5…`) | o insights mudou para os valores novos |
| Mudar só nome e telefone | a linha do insights ficou **igual** |
| Valores originais | restaurados nos dois bancos |
| Pedidos | 606 no ordering = 606 no insights |
| Ecommerce (conta nova, endereço, nome) e restart do ordering | **não rodados** (sem o front no ar) |

## Dados de teste

**Ecommerce (o roteiro do enunciado):**
1. Crie uma conta e complete o perfil → o cliente novo aparece no insights (`select * from customer order by registered_at desc limit 1`).
2. My account → Your Address: mude cidade e estado → os valores novos aparecem no insights.
3. My account → Account details: mude só nome e telefone → a linha do insights não muda.
4. Reinicie o ordering → o `afterMigrate.sql` apaga e recarrega os 22 clientes; o novo some, e a contagem volta a 22 (espere o sink terminar).

**Conferir no psql:**
```sql
-- insights (5434)
select count(*) from customer;
select count(id), count(birth_date), count(registered_at), count(address_city), count(archived_at) from customer;
```

## Onde ver

- `etc/kafka-connect/cdc.source.ordering.customer.json` e `cdc.sink.insights.customer.json` (meta).
- `etc/kafka-connect/cdc.sink.insights.order.json` (meta): a DLQ.
- `V002__create-table-customer.sql` (insights, `db/migration`).
