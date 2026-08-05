# Redis na prática

> O serviço no compose campo a campo, por que ele roda sem persistência, o que `allkeys-lru` decide quando a memória acaba — e a armadilha de interpolação que deixou o cache inteiro sem funcionar por horas sem nada acusar.
> Código real: `docker-compose.tools.yml`, `.env`, `application-development-env.yml` (product-catalog e ordering).

> Os padrões de cache — cache-aside, write-through, invalidação — estão em [`cache.md`](../01-arquitetura-design/cache.md). Aqui é a infraestrutura.

---

## O serviço

```yaml
# docker-compose.tools.yml
algashop-redis:
  image: redis:8.4
  restart: unless-stopped
  ports:
    - 6379:6379
  command: [
    "redis-server",
    "--requirepass", "${REDIS_PASSWORD}",
    "--appendonly", "no",
    "--save", "",
    "--maxmemory", "480mb",
    "--maxmemory-policy", "allkeys-lru"
  ]
  healthcheck:
    test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
  deploy:
    resources:
      limits:
        memory: 512M
```

Cada linha do `command` é uma decisão, e nenhuma é padrão.

### Sem persistência — de propósito

```
--appendonly no      desliga o AOF (log de cada escrita)
--save ""            desliga o RDB (snapshot periódico)
```

Redis sabe persistir, e aqui isso está **desligado nos dois modos**. A razão é o que este Redis é: um **cache**, não um banco.

Todo dado nele tem origem em outro lugar — Mongo, ou a API do catálogo. Perder o conteúdo custa uma janela de misses, não um dado. Em troca, some o custo de escrita em disco, some o risco de o AOF crescer sem controle, e o restart é instantâneo.

> O teste para saber se um Redis pode rodar assim: **se ele reiniciar vazio, alguma informação deixa de existir no sistema?** Se sim, ele não é um cache — e desligar persistência perde dado. Aqui a resposta é não.

Note que também **não há volume declarado**. É coerente: não haveria o que guardar.

### `maxmemory` abaixo do limite do container

```
--maxmemory 480mb        ← o que o Redis se permite usar com dados
memory: 512M             ← o que o Docker permite ao container
```

Os 32 MB de folga não são arredondamento. O Redis gasta memória além dos dados: buffers de cliente, buffers de replicação, fragmentação do alocador, a própria estrutura das chaves.

Se `maxmemory` fosse igual ao limite do container, o Redis acharia que ainda cabe e o **kernel** o mataria antes — `OOMKilled`, sem log, sem chance de aplicar a política de eviction. Com folga, quem decide o que sai é o Redis, e a decisão é ordenada.

### `allkeys-lru` — quem sai quando enche

Chegando ao `maxmemory`, o Redis precisa escolher. As políticas úteis:

| Política | Comportamento |
|---|---|
| `noeviction` (padrão) | **recusa escritas** com erro; leituras continuam |
| `volatile-lru` | despeja só chaves **com TTL**, a menos usada primeiro |
| `allkeys-lru` | despeja **qualquer** chave, a menos usada primeiro |
| `allkeys-lfu` | idem, mas pela **frequência** de uso |

`allkeys-lru` é a escolha certa para um cache puro: aqui toda chave é descartável, e a menos usada recentemente é a que menos falta faz.

> ⚠️ **O padrão do Redis é `noeviction`**, e ele é uma armadilha silenciosa num cache: ao encher, as escritas passam a falhar. Com o `CacheErrorHandler` engolindo erros — que é o desenho correto —, o sintoma seria a taxa de acerto despencando sem nada quebrar.

`allkeys-*` só é seguro porque **nada aqui é exclusivo do Redis**. Um Redis que também guardasse sessão ou fila precisaria de `volatile-*`, para o TTL marcar o que pode ser despejado.

---

## ⚠️ A armadilha que deixou o cache sem funcionar

O compose acima subiu por horas com o Redis **sem senha nenhuma**, enquanto as duas aplicações mandavam `password: algashop`. O `DBSIZE` ficou em zero o tempo todo, e nada acusou.

A causa está numa diferença de momento:

```yaml
command: ["redis-server", "--requirepass", "${REDIS_PASSWORD}"]   # ← Compose resolve
environment:
  REDIS_PASSWORD: algashop                                        # ← dentro do container
```

- **`${VAR}` em qualquer campo do YAML** é interpolado pelo **Compose**, ao ler o arquivo, a partir do ambiente do shell ou de um `.env`.
- **`environment:`** define a variável **dentro do container**, e só passa a existir depois que ele sobe.

O `environment:` não alimenta a interpolação. Com nenhum `.env` presente, o Compose resolveu `${REDIS_PASSWORD}` para **string vazia**:

```bash
docker compose -f docker-compose.tools.yml config
#   - --requirepass
#   - ""
```

E `requirepass ""` no Redis significa **desligar a autenticação**. Aí a aplicação manda `AUTH` num servidor que não tem senha configurada, e recebe:

```
ERR AUTH <password> called without any password configured for the default user.
```

A conexão falha, o `CacheErrorHandler` engole, e o serviço responde perfeitamente — só que sempre pelo banco.

**A correção é um `.env` na raiz do meta:**

```bash
REDIS_PASSWORD=algashop
```

Verificando os dois lados — só um não prova nada:

```bash
docker compose -f docker-compose.tools.yml config | grep -A1 requirepass   # tem que mostrar a senha

docker exec algashop-meta-algashop-redis-1 redis-cli -a algashop ping      # PONG
docker exec algashop-meta-algashop-redis-1 redis-cli ping                  # NOAUTH Authentication required.
```

> **A lição maior que a do Compose:** um cache mal configurado **não quebra nada**. A aplicação responde certo, os testes passam, e o único sintoma é performance que nunca melhorou. Diferente de um banco mal configurado, que falha alto. Por isso vale conferir o `DBSIZE` depois de subir — a evidência de que o cache funciona é haver chave nele.

---

## Inspecionando

```bash
docker exec -it algashop-meta-algashop-redis-1 redis-cli -a algashop
```

```
DBSIZE                          # quantas chaves no banco atual — o teste de fumaça
INFO keyspace                   # chaves por banco lógico, e quantas têm TTL
SELECT 1                        # troca para o banco do ordering (o catálogo usa o 0)

SCAN 0 MATCH algashop:* COUNT 100    # lista chaves sem travar o servidor
TTL algashop:products:v1:<uuid>      # segundos restantes; -1 = sem TTL, -2 = não existe

INFO memory                     # used_memory, maxmemory, fragmentação
CONFIG GET maxmemory-policy
INFO stats                      # keyspace_hits e keyspace_misses — a taxa de acerto
```

> ⚠️ **`KEYS *` percorre o keyspace inteiro e bloqueia o servidor** enquanto roda. Em desenvolvimento passa; o hábito é que é ruim. `SCAN` é incremental e não trava.

As chaves saem assim:

```
algashop:products:v1:19274f99-e0d2-40b1-9b3a-912cb0982f11
algashop:categories-filter:v1:default
```

Um dois-pontos entre o nome do cache e a chave, e não os dois do padrão do Spring — é o `computePrefixWith(c -> c + ":")` do `RedisCacheConfig`, escolhido para bater com a convenção usual de namespace.

E `GET` numa dessas chaves devolve **binário ilegível**: os valores são serializados pelo mecanismo nativo do Java, não em JSON. Ver [`cache.md`](../01-arquitetura-design/cache.md#serialização-jdk-não-json).

---

## A configuração do lado da aplicação

```yaml
# product-catalog — application-development-env.yml
spring:
  cache:
    type: redis
  data:
    redis:
      host: localhost
      port: 6379
      password: algashop
      database: 0        # o ordering usa o 1
      timeout: 600
```

Duas coisas para não errar:

**`spring.cache.type: redis` é o que liga tudo.** O `RedisCacheConfig` é `@ConditionalOnProperty` nessa chave, e é ele que traz o `@EnableCaching`. Sem a propriedade, a classe não é registrada, `@EnableCaching` não acontece, e **toda anotação de cache vira enfeite** — sem erro, sem aviso. É o que ocorre hoje nos perfis `docker` e `production`, que não a definem.

**`timeout: 600` são 600 milissegundos, não 600 segundos.** É um `Duration`, e sem unidade o Spring lê milissegundos. Para segundos, `600s`.

---

## Armadilhas

1. **`${VAR}` no compose vem do `.env`/shell, nunca do `environment:` do serviço.**
2. **`requirepass ""` desliga a autenticação** em vez de exigir senha vazia.
3. **Cache mal configurado não quebra nada** — só nunca acelera.
4. **`maxmemory` igual ao limite do container troca eviction ordenada por `OOMKilled`.**
5. **O padrão `noeviction` faz as escritas falharem** ao encher, em silêncio se houver `CacheErrorHandler`.
6. **`KEYS *` bloqueia o servidor.** Use `SCAN`.
7. **`timeout` sem unidade é milissegundos.**
8. **Sem `spring.cache.type: redis` não há cache** — e nada avisa.

---

## Pendências registradas

- [ ] **O cache não popula, mesmo com o Redis acessível.** Depois da correção da senha, `redis-cli -a algashop ping` responde e o servidor está saudável — mas com a aplicação de pé e servindo requisições, `DBSIZE` continua em `0` e `connected_clients` mostra que ela **nem abre conexão**. A causa está do lado da aplicação, não do Redis, e não foi identificada. Ver [`cache.md`](../01-arquitetura-design/cache.md#-o-estado-real-o-cache-não-popula-na-aplicação-rodando).
- [ ] **Um Redis para dois serviços.** Bancos lógicos separam namespace, não memória: o `maxmemory` é do processo inteiro, e o `allkeys-lru` despeja chaves de qualquer banco. Um serviço pode expulsar o cache do outro.
- [ ] **Sem TLS e sem ACL.** Senha única compartilhada, em texto no `.env` versionado. Aceitável localmente; num ambiente real seriam usuários por serviço com ACL restringindo comandos, e TLS no transporte.
- [ ] **`--requirepass` continua interpolado do `.env`.** Funciona, mas o `environment:` do serviço ficou lá sem uso — duas fontes para a mesma informação é o que causou o problema original.
- [ ] **Cache configurado só em `development`.** Nem `docker` nem `production` definem `spring.cache.type`.
- [ ] **Sem métrica de taxa de acerto.** `keyspace_hits`/`keyspace_misses` existem no `INFO`, mas nada os coleta — não há como saber se o cache está valendo a pena.
- [ ] **`billing` e `billing-scheduler` não usam cache.** Nada decidiu que não devem; simplesmente não foi avaliado.

---

## Checklist de revisão

- [ ] Sei dizer por que este Redis roda sem AOF e sem RDB
- [ ] Sei o teste para decidir se um Redis pode rodar sem persistência
- [ ] Entendo por que `maxmemory` fica abaixo do limite do container
- [ ] Sei o que cada política de eviction faz, e por que `allkeys-lru` serve aqui
- [ ] Entendo por que `noeviction` é perigoso num cache
- [ ] Sei explicar por que `${VAR}` no `command` não vê o `environment:`
- [ ] Sei conferir se o cache está de fato funcionando, e não só se o serviço responde
- [ ] Sei por que `SCAN` e não `KEYS`
- [ ] Sei que `timeout: 600` são 600 ms

---

## Referências

- [Redis — Key eviction](https://redis.io/docs/latest/develop/reference/eviction/)
- [Redis — Persistence](https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/)
- [Redis — `SCAN`](https://redis.io/docs/latest/commands/scan/)
- [Docker Compose — Variable interpolation](https://docs.docker.com/compose/how-tos/environment-variables/variable-interpolation/)
- [Spring Boot — Caching](https://docs.spring.io/spring-boot/reference/io/caching.html)
- [`cache.md`](../01-arquitetura-design/cache.md) — os padrões e a invalidação
- [`ambiente-local.md`](./ambiente-local.md) — portas, bancos e problemas comuns
- [`docker.md`](./docker.md) — build de imagem e o compose de apoio
