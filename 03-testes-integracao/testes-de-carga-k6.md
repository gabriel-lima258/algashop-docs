# Testes de carga com k6

> As dezessete fases anteriores mediram **correção**: o pedido fecha, o evento propaga, o circuito abre, a transação faz rollback. Nenhuma mediu **capacidade**. Este documento é sobre o instrumento — e sobre as três armadilhas que fazem um teste de carga medir menos do que ele diz medir.
> Código real: `etc/k6/get-product-loadTest-k6.js` e `etc/k6/buy-now.js`, no repositório meta.

> Os números medidos e o que eles revelaram sobre threads estão em [Threads e concorrência](../04-infraestrutura/threads-e-concorrencia.md). Aqui fica a ferramenta.

---

## O que cada tipo de teste pergunta

Todos usam a mesma ferramenta, e a diferença é inteiramente a **pergunta**. Confundir os quatro é o erro mais comum, porque os quatro se parecem na tela.

| Tipo | Pergunta | Como se parece |
|---|---|---|
| **Smoke** | funciona? | 1 VU, poucos segundos |
| **Load** | aguenta o que a gente espera? | taxa fixa e realista, alguns minutos |
| **Stress / volume** | aguenta **até onde**? | rampa crescente até doer |
| **Soak** | aguenta por **quanto tempo**? | carga moderada por horas |

O smoke não é um teste de carga menor — é um **pré-requisito**. Rodar uma hora de carga contra uma API que devolve 500 é uma hora perdida, e o smoke custa 5 segundos. Por isso os dois scripts começam com ele:

```js
const smokeScenario = {
  executor: 'constant-vus',
  vus: 1,
  duration: '5s',
  exec: 'buyNowWithThinkTime',
};
```

E o `startTime: '5s'` do cenário seguinte garante que os dois **nunca se sobrepõem**: o load começa exatamente quando o smoke termina.

---

## Executores: VU não é taxa

Esta é a distinção que decide se o seu teste mede alguma coisa.

**`constant-vus`** — N usuários virtuais em laço. VU é **concorrência**: quantas requisições existem ao mesmo tempo. Se a API ficar lenta, cada VU completa menos iterações e **a carga cai sozinha**.

> É por isso que `constant-vus` é ruim para medir capacidade: o teste alivia exatamente quando o sistema aperta. Ele mede o comportamento sob uma concorrência, não sob uma demanda.

**`constant-arrival-rate`** — N requisições por segundo, doa a quem doer. A taxa **não** cai quando a API fica lenta; o k6 aloca mais VUs para sustentá-la. É o único que reproduz o mundo real, onde os clientes continuam chegando quando o sistema engasga.

**`ramping-vus`** — a concorrência sobe em degraus. É o executor do teste de volume: a pergunta não é "aguenta?", é "aguenta até onde?".

**`ramping-arrival-rate`** — a taxa sobe em degraus. É o que encontra o joelho da curva de capacidade, e foi o que localizou o teto deste sistema.

A regra prática:

| Você quer saber | Executor |
|---|---|
| como se comporta com N usuários simultâneos | `constant-vus` |
| se aguenta N requisições por segundo | `constant-arrival-rate` |
| em que concorrência quebra | `ramping-vus` |
| em que **taxa** quebra | `ramping-arrival-rate` |

---

## Armadilha 1 — `sleep()` no executor errado

Os dois scripts nasceram assim:

```js
export default function() {
  let res = http.get(url);
  check(res, { "status is 200": (res) => res.status === 200 });
  sleep(1);              // <- dentro de um cenário constant-arrival-rate
}
```

`sleep()` modela o tempo em que a pessoa fica olhando a tela. Ele faz todo sentido em `constant-vus`, onde o VU é uma pessoa. Em `constant-arrival-rate` ele **não segura nada** — quem dita o ritmo é o executor. O que ele faz é aumentar a duração de cada iteração em 1 segundo.

E isso importa porque o k6 dimensiona VUs assim:

```
VUs necessários = taxa × duração da iteração
```

A 100 req/s com iteração de ~1s, são **100 VUs só por causa do sleep**. O `maxVUs` era exatamente 100. Zero folga. Quando falta VU, o k6 não avisa em vermelho: ele **descarta iterações** e segue.

A correção separou as duas coisas em duas funções, em vez de um `if` dentro do request — porque o sleep não é detalhe da requisição, é o que distingue os dois executores:

```js
export function buyNow() {
  const res = http.post(`${BASE_URL}/api/v1/orders`, body, params);
  check(res, { 'status is 201': (r) => r.status === 201 });
}

export function buyNowWithThinkTime() {
  buyNow();
  sleep(1);
}
```

E cada cenário aponta para a sua com `exec:`.

### `dropped_iterations` é o alarme

```js
dropped_iterations: ['count < 1'],
```

Sem essa linha, um teste que aplicou metade da carga que declarou termina **verde**. Com ela, faltar VU vira falha.

---

## Armadilha 2 — `check()` não reprova nada

Esta é a que mais engana, porque a saída do k6 mostra os checks com ✓ e ✗ bem visíveis.

> **`check()` só conta.** Um script com 100% de checks vermelhos ainda termina com **exit code 0**. O único mecanismo que reprova um teste é o **threshold**.

O `buy-now.js` declarava apenas:

```js
thresholds: {
  http_req_duration: ['p(95) < 1200']
}
```

Ou seja: ele passava com a API devolvendo **422 em todas as requisições**, desde que devolvesse rápido. E erro costuma ser mais rápido que sucesso — a validação falha antes de tocar o banco. O teste ficava *mais verde* quanto pior a API estivesse.

```js
'http_req_failed{scenario:buy_now_load}': ['rate < 0.01'],
'checks{scenario:buy_now_load}': ['rate > 0.99'],
```

Falhar depressa não é passar.

---

## Armadilha 3 — o smoke diluindo o load

Threshold sem filtro é calculado sobre **todas** as amostras do teste. Com o smoke e o load no mesmo script, os 5 segundos de 1 VU entram na mesma amostra do minuto a 100 req/s e puxam o percentil para baixo.

A correção é o filtro por tag — `scenario` é uma tag de sistema, já existe:

```js
'http_req_duration{scenario:get_product_load}': ['p(95) < 800'],
```

Assim o limite pergunta sobre o cenário que interessa, não sobre a média de dois regimes diferentes.

---

## Dimensionar `preAllocatedVUs` e `maxVUs`

```js
const loadScenario = {
  executor: 'constant-arrival-rate',
  rate: 80,
  timeUnit: '1s',
  duration: '1m',
  startTime: '5s',
  preAllocatedVUs: 96,
  maxVUs: 250,
  exec: 'buyNow',
};
```

A conta é a mesma de sempre: `VUs = taxa × duração da iteração`. A 80/s com o `p(95)` exigido de 1200ms, o **pior caso** pede 80 × 1,2 = 96 VUs.

`maxVUs` precisa de folga acima disso, e por um motivo específico: quando a API degrada além do alvo, a iteração dura mais e a demanda de VUs sobe. Sem folga, o k6 para de aplicar carga **justamente na hora interessante** — e o teste sai verde por não ter conseguido apertar.

> Na prática medida, a listagem de produtos responde em 2,6ms. A 100 req/s isso dá 0,26 VU — o teste rodou com **1 VU ativo** enquanto 80 estavam pré-alocados. Pré-alocar demais custa memória no k6, não carga no alvo; pré-alocar de menos custa latência de rampa. Errar para cima é barato.

---

## Ler o resultado

```
http_req_duration: avg=6.99ms min=3.36ms med=6.06ms max=196.28ms p(90)=9.23ms p(95)=11.4ms
```

**A média mente.** Ela dilui a cauda: `avg=6,99ms` com `max=196ms` significa que alguém esperou 28 vezes mais que a média, e a média não conta isso. É a cauda que o usuário percebe e que derruba SLA.

O que ler, em ordem:

| Métrica | O que ela responde |
|---|---|
| `http_reqs` /s | a taxa que o sistema **realmente** sustentou |
| `dropped_iterations` | se o teste conseguiu aplicar a carga que prometeu |
| `p(95)` / `p(99)` | o que a cauda sofreu |
| `http_req_failed` | quanto quebrou |
| `vus` (max) | quanta concorrência foi precisa — compare com o teto do servidor |
| `iteration_duration` vs `http_req_duration` | a diferença é `sleep` + overhead do script |

O par `http_reqs/s` **muito abaixo** da taxa pedida junto com `dropped_iterations` alto é a assinatura da saturação: o alvo não deu conta e o gerador não conseguiu insistir.

---

## Perfis num script só

Em vez de deixar o teste de volume comentado no topo do arquivo — onde ele apodrece —, os cenários viram objetos e o perfil escolhe:

```js
const PROFILE = __ENV.PROFILE || 'default';

export const options = {
  scenarios: PROFILE === 'volume'
    ? { buy_now_volume: volumeScenario }
    : { buy_now_smoke: smokeScenario, buy_now_load: loadScenario },
  thresholds: PROFILE === 'volume' ? {} : { /* ... */ },
};
```

O teste de volume **não herda os thresholds**: falhar faz parte do resultado dele. Um limite de `p(95)` num teste cuja finalidade é encontrar o ponto de quebra não mede nada — ele só garante vermelho.

```bash
k6 run etc/k6/buy-now.js
k6 run -e PROFILE=volume etc/k6/buy-now.js
```

### Duas pegadinhas de `__ENV`, as duas silenciosas

1. **Não use o prefixo `K6_`** no nome da variável. `K6_*` é o namespace de configuração do próprio k6 — ele consome essas variáveis como opções e elas nunca chegam em `__ENV`. A primeira versão usava `K6_PROFILE` e o perfil `volume` simplesmente não existia.
2. **Passe com `-e`, não pelo ambiente do shell.** No k6 v2 as variáveis do sistema não entram em `__ENV` por padrão. `PROFILE=volume k6 run ...` roda o perfil `default` **calado**, sem erro nenhum.

As duas falham do mesmo jeito: o teste roda, sai verde, e mediu outra coisa.

---

## Os dois cenários deste projeto

| Script | Endpoint | Por que existe |
|---|---|---|
| `get-product-loadTest-k6.js` | `GET /api/v1/products` (catálogo, 8083) | leitura pura servida pelo cache. É o **piso** de comparação |
| `buy-now.js` | `POST /api/v1/orders` (ordering, 8081) | o caminho mais caro: transação no Postgres + duas chamadas HTTP de saída |

A diferença entre os dois é o ponto: no primeiro o gargalo é CPU e serialização; no segundo é **thread parada esperando rede**, dentro de uma transação aberta.

Os ids são fixos e vêm carregados — o cliente pelo `afterMigrate.sql` do Flyway, o produto pelo `products.json` do `DataLoader` do catálogo. Todo VU compra o mesmo produto para o mesmo cliente, o que concentra contenção no mesmo registro. É deliberado: é o pior caso.

> ⚠️ O `customerId` fixo é o cliente **anonimizado** (`archived = true`, 0 pontos de fidelidade). O teste percorre um caminho que nenhum usuário real percorre — em particular, nunca cai na regra de frete grátis. Números de latência valem; a distribuição de caminhos, não.

---

## Como rodar

O alvo precisa estar de pé:

```bash
docker compose up -d
```

```bash
k6 run etc/k6/get-product-loadTest-k6.js
k6 run etc/k6/buy-now.js
k6 run -e PROFILE=volume etc/k6/buy-now.js
```

Apontando para outro host:

```bash
k6 run -e BASE_URL=http://localhost:9090 etc/k6/buy-now.js
```

---

## Pendências

- **Nenhum resultado é guardado.** Cada rodada existe só no terminal. Sem série histórica não há como detectar regressão de performance — só dá para comparar com a memória de quem rodou. Um `--out json` versionado, ou `handleSummary()` gravando um resumo, resolveria.
- **Não roda em CI**, e provavelmente não deveria rodar como está: os números dependem da máquina. Teste de carga em CI precisa de alvo dedicado, senão vira gerador de falso positivo.
- **Os dois scripts escrevem no banco** (`buy-now` cria pedido a cada iteração) e nada limpa depois. Uma rodada de volume deixa dezenas de milhares de pedidos. Rodadas sucessivas medem um banco cada vez maior.
- **Sem cenário de leitura no `ordering`.** Só há escrita. O `GET /api/v1/orders` sobre uma tabela que cresce a cada rodada é justamente onde a paginação e os índices apareceriam.
- **Os thresholds foram escolhidos por intuição** (800ms e 1200ms), não a partir de um requisito. Medido, o `p(95)` real é 4,9ms e 11,3ms — os limites estão duas ordens de grandeza acima e não reprovariam nem uma regressão de 100×.

---

## Checklist de revisão

- [ ] Todo cenário de `arrival-rate` está **sem `sleep()`**?
- [ ] Existe threshold de `http_req_failed`, e não só de duração?
- [ ] Existe threshold de `dropped_iterations`?
- [ ] Os thresholds estão filtrados por `{scenario:...}` quando há mais de um cenário?
- [ ] `maxVUs` tem folga sobre `taxa × duração_da_iteração` no pior caso?
- [ ] O teste de volume está **livre** de thresholds?
- [ ] As variáveis de ambiente estão sem o prefixo `K6_` e são passadas com `-e`?
- [ ] O smoke roda antes, e os cenários não se sobrepõem?

---

## Referências

- [k6 — Executors](https://grafana.com/docs/k6/latest/using-k6/scenarios/executors/)
- [k6 — Thresholds](https://grafana.com/docs/k6/latest/using-k6/thresholds/)
- [k6 — Scenarios](https://grafana.com/docs/k6/latest/using-k6/scenarios/)
- [Threads e concorrência](../04-infraestrutura/threads-e-concorrencia.md) — o que estas medições revelaram
