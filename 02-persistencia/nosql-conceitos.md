# Resiliência, Escalabilidade e NoSQL

> Material de estudo consolidado — Módulo 1
> Cobre os tópicos 1.1 a 1.7 do conteúdo programático.

---

## 1.1. Introdução ao Nível

Este módulo introduz três conceitos fundamentais que se conectam diretamente em sistemas modernos:

- **Resiliência**: capacidade de um sistema continuar operando (mesmo que de forma degradada) diante de falhas — quedas de nó, partições de rede, picos de carga.
- **Escalabilidade**: capacidade de crescer (em volume de dados, requisições ou usuários) sem reescrever o sistema. Pode ser **vertical** (mais CPU/RAM em uma máquina) ou **horizontal** (mais máquinas).
- **NoSQL**: famílias de bancos de dados criadas para resolver limitações que bancos relacionais enfrentam em cenários de escala massiva e dados não-estruturados.

A relação entre os três: bancos NoSQL surgiram justamente como resposta à dificuldade dos relacionais de escalar horizontalmente mantendo resiliência em ambientes distribuídos.

---

## 1.2. Bancos de Dados Relacionais e os Desafios Enfrentados

### Características dos relacionais (SGBDRs)

- Dados organizados em **tabelas** com schema fixo (linhas e colunas tipadas)
- Relacionamentos via **chaves estrangeiras**
- Linguagem padrão: **SQL**
- Garantias **ACID** (Atomicidade, Consistência, Isolamento, Durabilidade)
- Otimizados para **transações** e integridade referencial

**Exemplos:** PostgreSQL, MySQL, Oracle, SQL Server.

### Desafios em escala

1. **Escala horizontal difícil**
    - Sharding manual é complexo (definir chave de partição, lidar com queries cross-shard)
    - JOINs entre shards praticamente inviáveis em performance aceitável

2. **Schema rígido**
    - Alterações de schema em tabelas grandes podem travar a aplicação
    - Migrations exigem disciplina e janelas de manutenção

3. **Modelo relacional nem sempre encaixa**
    - Dados hierárquicos (árvores, documentos aninhados) exigem múltiplas tabelas e JOINs
    - Dados sem estrutura fixa (logs, eventos) ficam artificiais em tabelas

4. **Custo de consistência forte**
    - Transações distribuídas (2PC) são caras e travam recursos
    - Bloqueios de linha/tabela limitam throughput sob alta concorrência

5. **Limites de hardware vertical**
    - Eventualmente não dá pra comprar uma máquina maior

> **Observação importante:** isso **não significa que relacionais sejam ruins**. Para a maioria absoluta de sistemas (incluindo SaaS de porte médio), PostgreSQL bem modelado resolve. NoSQL entra quando o padrão de acesso ou a escala genuinamente justifica.

---

## 1.3. Bancos de Dados Não Relacionais (NoSQL)

### O que é NoSQL?

O termo significa **"Not Only SQL"** — não é "anti-SQL", é "além de SQL". Refere-se a uma família heterogênea de bancos que abandonam total ou parcialmente o modelo relacional para otimizar outros aspectos: escala horizontal, flexibilidade de schema, performance em padrões específicos.

### Características gerais

- **Schema flexível ou ausente** (schema-on-read em vez de schema-on-write)
- **Escala horizontal nativa** (sharding e replicação como cidadãos de primeira classe)
- **Modelo de dados específico** para o caso de uso (documento, chave-valor, grafo, etc.)
- **Geralmente sacrificam algo de ACID** em favor de disponibilidade e partição (ver Teorema CAP)
- **APIs próprias** em vez de SQL universal (embora muitos hoje suportem dialetos SQL-like)

### Quando NoSQL faz sentido

- Volume de dados que excede capacidade de uma instância relacional
- Padrão de acesso conhecido e específico (não exploratório)
- Dados naturalmente não-tabulares (documentos, grafos, séries temporais)
- Necessidade de baixa latência em escala global
- Schema que evolui rapidamente durante desenvolvimento

### Bancos NoSQL mais utilizados

- **MongoDB** — líder absoluto entre bancos NoSQL primários
- **Redis** — onipresente em stacks como cache, broker e estruturas auxiliares
- **Cassandra/ScyllaDB** — dominantes em escala massiva
- **Elasticsearch** — padrão de facto para full-text search
- **DynamoDB** — referência em key-value gerenciado (AWS)

> MongoDB lidera consistentemente o **DB-Engines Ranking** entre os NoSQL. Redis frequentemente aparece em primeiro em pesquisas como o **Stack Overflow Developer Survey** quando a pergunta envolve "bancos mais amados/usados", mas geralmente como componente complementar, não como banco primário.

---

## 1.4. As Famílias de Bancos de Dados Não Relacionais

Existem **quatro famílias clássicas**, e algumas categorias adicionais que ganharam relevância nos últimos anos.

### 1. Key-Value Stores (Chave-Valor)

**Modelo:** uma chave única mapeia para um valor opaco. Estrutura mais simples possível.

**Exemplos:** Redis, DynamoDB, Memcached, Riak, etcd.

**Casos de uso:**
- Cache de aplicação
- Sessões web
- Feature flags e configurações
- Rate limiting e contadores
- Leaderboards em tempo real (Redis)

### 2. Document Stores (Documentos)

**Modelo:** documentos semiestruturados (JSON/BSON), cada um autocontido e potencialmente com schema diferente.

**Exemplos:** MongoDB, Couchbase, Firestore, Amazon DocumentDB.

**Casos de uso:**
- Catálogos de produtos com atributos variáveis
- CMS e gerenciamento de conteúdo
- Perfis de usuário com campos opcionais
- Event sourcing e logs estruturados
- APIs que trabalham nativamente com JSON

### 3. Column-Family Stores (Wide-Column)

**Modelo:** dados organizados em famílias de colunas, fisicamente armazenados por coluna. Otimizados para escrita massiva e leitura de poucas colunas em muitas linhas.

**Exemplos:** Cassandra, ScyllaDB, HBase, Google Bigtable.

**Casos de uso:**
- Séries temporais em larga escala (IoT, métricas)
- Logs de eventos distribuídos
- Histórico de mensagens (chat, notificações)
- Analytics OLAP
- Aplicações geo-distribuídas com altíssima escrita

### 4. Graph Databases (Grafos)

**Modelo:** nós (entidades) e arestas (relacionamentos), ambos com propriedades. Traversals de relacionamentos são O(1) por aresta.

**Exemplos:** Neo4j, Amazon Neptune, ArangoDB, Dgraph.

**Casos de uso:**
- Redes sociais
- Detecção de fraude (análise de conexões)
- Sistemas de recomendação relacionais
- Knowledge graphs
- Análise de impacto em redes/dependências

### Famílias adicionais

| Família | Exemplos | Caso de uso principal |
|---|---|---|
| **Search Engines** | Elasticsearch, OpenSearch | Full-text search, filtros facetados |
| **Time-Series** | InfluxDB, TimescaleDB, Prometheus | Métricas, monitoramento, IoT |
| **Vector** | Pinecone, Weaviate, Qdrant, pgvector | Embeddings, RAG, busca semântica |

---

## 1.5. Como os Diferentes Bancos Consultam os Dados?

A forma de consultar varia drasticamente entre famílias — não há padrão universal como SQL nos relacionais.

### Relacionais — SQL

Linguagem declarativa universal. O programador descreve **o que** quer, o otimizador decide **como** buscar.

```sql
SELECT u.nome, COUNT(p.id) AS total_pedidos
FROM usuarios u
JOIN pedidos p ON p.usuario_id = u.id
WHERE u.status = 'ATIVO'
GROUP BY u.nome;
```

### Key-Value — Acesso direto por chave

Sem linguagem de query — apenas operações primitivas: `GET`, `SET`, `DEL`, `INCR`.

```
GET session:abc123
SET cache:produto:42 "{...json...}" EX 3600
INCR contador:requisicoes:2026-05-15
```

Não há filtros, JOINs ou agregações nativas. Para consultas complexas, a aplicação precisa manter índices secundários manualmente.

### Document Stores — Linguagens próprias

MongoDB usa uma API baseada em JSON:

```javascript
db.pedidos.find({
  status: "PAGO",
  total: { $gte: 100 }
}).sort({ criadoEm: -1 }).limit(10)

db.pedidos.aggregate([
  { $match: { status: "PAGO" } },
  { $group: { _id: "$clienteId", total: { $sum: "$valor" } } }
])
```

Permite filtros, projeções, agregações (pipeline) e índices em campos aninhados.

### Column-Family — CQL e similares

Cassandra usa **CQL** (Cassandra Query Language), sintaticamente parecido com SQL mas com restrições importantes: sem JOINs, sem subqueries, e queries precisam respeitar a chave de partição.

```sql
SELECT * FROM eventos
WHERE usuario_id = '123'
  AND data >= '2026-05-01';
```

### Graph — Linguagens de traversal

Neo4j usa **Cypher**:

```cypher
MATCH (u:Usuario {nome: 'Gabriel'})-[:AMIGO_DE*1..2]->(amigo)
RETURN DISTINCT amigo.nome
```

A query acima encontra amigos de amigos (até 2 níveis) — algo extremamente caro em SQL com múltiplos JOINs recursivos.

### Search Engines — Query DSL

Elasticsearch usa uma DSL JSON:

```json
{
  "query": {
    "bool": {
      "must": [
        { "match": { "titulo": "contrato" } },
        { "range": { "data": { "gte": "2026-01-01" } } }
      ]
    }
  }
}
```

### Resumo comparativo

| Família | Linguagem | Suporta JOIN? | Suporta agregações? |
|---|---|---|---|
| Relacional | SQL | Sim | Sim |
| Key-Value | API primitiva | Não | Não |
| Document | API JSON / pipelines | Limitado (`$lookup`) | Sim |
| Column-Family | CQL | Não | Limitado |
| Graph | Cypher / Gremlin | Via traversal | Sim |
| Search | Query DSL | Não | Sim (agregações) |

---

## 1.6. Estrutura, Dados e Esquema

### Schema-on-Write vs Schema-on-Read

**Schema-on-Write (relacionais)**
- O schema é definido **antes** da escrita
- Banco valida tipos, constraints, FKs no momento do `INSERT`
- Garante consistência estrutural
- Custo: alterações exigem migrations

**Schema-on-Read (muitos NoSQL)**
- O schema é **interpretado pela aplicação** no momento da leitura
- Banco aceita qualquer estrutura
- Flexibilidade máxima
- Custo: a aplicação precisa lidar com variações e versões

### Estruturas de dados típicas

| Família | Estrutura | Exemplo |
|---|---|---|
| Relacional | Tabela (linhas × colunas) | `usuarios(id, nome, email)` |
| Key-Value | Par chave-valor | `"user:42" → {...}` |
| Document | Documento aninhado | `{ id, nome, enderecos: [{...}] }` |
| Column-Family | Linha com colunas dinâmicas | `partition_key → {col1, col2, ...}` |
| Graph | Nós + arestas | `(Usuario)-[COMPROU]->(Produto)` |

### Implicações práticas

- **Normalização vs Desnormalização**: relacionais favorecem normalização (3FN); NoSQL geralmente favorece desnormalização (duplicar dados para evitar JOINs).
- **Versionamento de schema**: em document stores, é comum guardar um campo `schemaVersion` e migrar dados gradualmente em vez de uma migration única.
- **Validação**: MongoDB e outros permitem schemas opcionais (JSON Schema validators) — flexibilidade quando você quer, rigor quando precisa.

---

## 1.7. ACID e BASE

São dois conjuntos de garantias **opostos em filosofia**, refletindo escolhas de design entre consistência e disponibilidade.

### ACID — Bancos Relacionais

| Letra | Significado | Descrição |
|---|---|---|
| **A** | Atomicidade | Transação executa por completo ou não executa nada. Sem estados intermediários. |
| **C** | Consistência | Toda transação leva o banco de um estado válido para outro estado válido (respeitando constraints). |
| **I** | Isolamento | Transações concorrentes não interferem entre si. Níveis: Read Uncommitted, Read Committed, Repeatable Read, Serializable. |
| **D** | Durabilidade | Uma vez commitada, a transação persiste — mesmo que o sistema caia logo depois. |

**Filosofia:** prefiro estar correto a estar disponível. Se houver dúvida, falho.

### BASE — Muitos Bancos NoSQL

| Sigla | Significado |
|---|---|
| **BA** | **Basically Available** — o sistema está disponível na maior parte do tempo, mesmo que com dados desatualizados |
| **S** | **Soft state** — o estado pode mudar com o tempo, mesmo sem novas escritas (devido a replicação assíncrona) |
| **E** | **Eventually consistent** — eventualmente, dados replicados convergem para o mesmo valor |

**Filosofia:** prefiro estar disponível a estar perfeitamente consistente. Aceito inconsistência temporária em troca de uptime e escala.

### Teorema CAP (contexto fundamental)

Em sistemas distribuídos, você só pode garantir **2 das 3** propriedades simultaneamente:

- **C**onsistência (todos os nós veem o mesmo dado ao mesmo tempo)
- **A**vailability (cada requisição recebe resposta — sucesso ou falha)
- **P**artition tolerance (sistema continua funcionando apesar de falhas de rede entre nós)

Como **partições de rede são inevitáveis** em sistemas distribuídos reais, na prática a escolha é entre **CP** (sacrifica disponibilidade) e **AP** (sacrifica consistência forte).

| Tipo | Exemplos |
|---|---|
| **CP** (consistente + tolerante a partição) | MongoDB (configurado para isso), HBase, etcd |
| **AP** (disponível + tolerante a partição) | Cassandra, DynamoDB, Couchbase |
| **CA** (apenas em sistemas não distribuídos) | PostgreSQL standalone, MySQL single-node |

### ACID vs BASE — Quando usar cada um?

**Use ACID quando:**
- Dinheiro está envolvido (cobrança, billing, contabilidade)
- Inconsistência tem custo legal ou regulatório
- O domínio exige garantias fortes (estoque, reservas, votação)

**Use BASE quando:**
- Disponibilidade é mais crítica que consistência imediata
- Inconsistência temporária é tolerável (likes em rede social, contadores de visualização)
- Escala global com baixa latência é requisito

> Importante: muitos NoSQL modernos (MongoDB 4+, DynamoDB) hoje oferecem transações ACID multi-documento. A divisão ACID/BASE não é mais tão binária quanto era em 2010.

---

## Resumo Geral do Módulo

1. Sistemas modernos precisam ser **resilientes** (sobreviver a falhas) e **escaláveis** (crescer sem reescrita).
2. **Bancos relacionais** são excelentes para a maioria dos casos, mas têm limitações em escala horizontal e flexibilidade de schema.
3. **NoSQL** ("Not Only SQL") é uma família heterogênea de bancos otimizados para casos onde relacionais sofrem.
4. **Quatro famílias clássicas**: Key-Value, Document, Column-Family, Graph — além de Search, Time-Series e Vector.
5. **Linguagens de consulta** variam radicalmente — não existe SQL universal no mundo NoSQL.
6. **Schema-on-Write** (relacionais) traz rigor; **Schema-on-Read** (NoSQL) traz flexibilidade.
7. **ACID** prioriza consistência; **BASE** prioriza disponibilidade. O **Teorema CAP** explica por quê a escolha é necessária em sistemas distribuídos.
8. Na prática, arquiteturas modernas usam **polyglot persistence**: combinam vários bancos, cada um onde brilha.