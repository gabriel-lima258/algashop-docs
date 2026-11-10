# AlgaShop — Portfólio de Estudos

E-commerce construído em microsserviços com **Java 25 + Spring Boot 4**, como projeto de estudo de arquitetura de software.

Este repositório é o caderno do projeto: cada documento registra um conceito aplicado, o problema que ele resolve, o código real onde ele aparece e as armadilhas encontradas no caminho.

---

## Os serviços

| Serviço | Porta | Banco | Destaque técnico |
|---|---|---|---|
| **`algashop-ordering`** | 8081 | PostgreSQL + Redis | DDD tático, arquitetura hexagonal, CQRS, domain events, cache client-side, resiliência |
| **`algashop-billing`** | 8082 | PostgreSQL | Integração com gateway de pagamento, resiliência por idempotência, Testcontainers |
| **`product-catalog`** | 8083 | MongoDB (replica set) + Redis | Modelagem documental e desnormalização, eventos de domínio, concorrência atômica, transação multi-coleção, cache server-side, REST Docs |
| **`notifications`** | 8084 | PostgreSQL | E-mail por templates: o ordering decide *quando*, ele decide *como*; HTML do template validado na entrada (jsoup) renderizado com Thymeleaf e enviado por SMTP; APIs de cadastro de templates e de envio protegidas por JWT e escopo — em construção |
| **`billing-scheduler`** | — | — | Jobs agendados, cancelamento de faturas expiradas |
| **`authorization-server`** | 9000 | PostgreSQL | OAuth 2.1 e OIDC, gestão de usuários, PKCE para cliente público, sessão em banco |
| **`service-registry`** | 8761 | — | Eureka Server: registro e descoberta de serviços, dashboard das instâncias |
| **`api-gateway-ecommerce`** | 9999 | Redis (rate limit, db 3) | Borda do e-commerce: rotas por service ID, token na borda, cache local, API composition da home e resiliência completa |
| **`api-gateway-admin`** | 9998 | Redis (rate limit, db 5) | Borda do admin: CORS da SPA, JSON enxuto na listagem (`RemoveJsonAttributesResponseBody`), mesmas fundações de resiliência |
| **`ecommerce-app`** | 9080 | Redis (sessão, db 4) | O BFF: Spring + Thymeleaf, OAuth2 client confidencial, token em sessão — o navegador nunca o vê |
| **`admin-app`** | 4200 | — | SPA Angular 17 com PKCE, falando direto com o authorization server para login |

> Como os serviços se conectam, quem chama quem e por quê: **[Arquitetura](./00-visao-geral/01-arquitetura.md)**

---

## Trilha de leitura

Cada pasta é uma etapa, e **dentro de cada pasta os arquivos são numerados na ordem em que devem ser lidos** (`01-…`, `02-…`). Siga as pastas de 00 a 07; dentro de cada uma, siga os números.

> 💡 Quer rodar o projeto antes de estudar? Leia [Ambiente local](./04-infraestrutura/01-ambiente-local.md) logo depois da visão geral.

### 00 — Visão geral

Comece aqui: o mapa do sistema e a história de como ele foi construído.

| # | Documento | O que você aprende |
|---|---|---|
| 01 | [Arquitetura](./00-visao-geral/01-arquitetura.md) | Mapa dos serviços, comunicação entre eles, persistência poliglota e os princípios que se repetem |
| 02 | [Linha do tempo](./00-visao-geral/02-linha-do-tempo.md) | A jornada em 45 fases — o que foi construído em cada etapa e por que naquela ordem |

### 01 — Arquitetura e design

Como o código de cada serviço é organizado por dentro.

| # | Documento | O que você aprende |
|---|---|---|
| 01 | [Ports & Adapters (Hexagonal)](./01-arquitetura-design/01-ports-hexagonal.md) | Por que separar `ports/in` de `ports/out`, quem implementa e quem consome cada porta |
| 02 | [CQS e CQRS](./01-arquitetura-design/02-cqrs.md) | Separar comando de consulta, as abordagens de CQRS e o que **não** confundir |
| 03 | [Specification Pattern](./01-arquitetura-design/03-specification.md) | Transformar regra de negócio em objeto combinável, em vez de `if` aninhado |
| 04 | [Eventos e listeners](./01-arquitetura-design/04-eventos-e-listeners.md) | Evento de domínio × de aplicação, `AbstractAggregateRoot`, `@Async` e o que a consistência eventual custa |
| 05 | [Cache](./01-arquitetura-design/05-cache.md) | Cache-aside × write-through, client-side × server-side, invalidação — e por que a idade de um dado é a **soma** das camadas |
| 06 | [Resiliência](./01-arquitetura-design/06-resiliencia.md) | Timeout, retry, bulkhead, circuit breaker e fallback — a ordem em que se aninham, e quando um fallback mente |

### 02 — Persistência

Como os dados são guardados e consultados — do Postgres com Flyway ao MongoDB.

| # | Documento | O que você aprende |
|---|---|---|
| 01 | [Flyway](./02-persistencia/01-flyway.md) | Versionar o schema como código; por que `ddl-auto` não serve para produção |
| 02 | [Paginação](./02-persistencia/02-paginacao.md) | Paginação com Criteria API, projeção com `builder.construct()` e consulta de contagem |
| 03 | [NoSQL — conceitos](./02-persistencia/03-nosql-conceitos.md) | Resiliência, escalabilidade, teorema CAP e as famílias de banco NoSQL |
| 04 | [MongoDB no product-catalog](./02-persistencia/04-product-catalog-mongo.md) | Modelagem documental, embutir vs. referenciar, UUID como `_id`, auditoria e lock otimista |
| 05 | [Consultas dinâmicas com Criteria](./02-persistencia/05-consultas-mongo-criteria.md) | Filtro com N parâmetros opcionais, `$expr`, busca textual e paginação manual no Mongo |
| 06 | [Índices no MongoDB](./02-persistencia/06-indices-mongo.md) | `explain`, regra ESR, índice composto, índice parcial, índice de texto e ordenação por relevância |
| 07 | [Aggregation Pipeline](./02-persistencia/07-agregacoes-mongo.md) | O outro jeito de consultar: `$lookup` contra o N+1, campos derivados no `$project` e por que a ordem dos estágios é o custo |
| 08 | [Normalizado × desnormalizado](./02-persistencia/08-desnormalizacao-mongo.md) | Quando duplicar dado é decisão e não bagunça — a cópia embutida, o que ela cobra e a pegadinha do `category._id` |
| 09 | [Concorrência e atomicidade](./02-persistencia/09-concorrencia-e-atomicidade.md) | Lost update, atualização condicional com `findAndModify`, `$inc` como delta e por que teste sequencial não prova nada |
| 10 | [Transações e replica set](./02-persistencia/10-transacoes-mongo.md) | Por que transação no Mongo exige um cluster, o que o `MongoTransactionManager` liga — e quando transação **não** acrescenta nada |
| 11 | [Armazenamento de arquivos](./02-persistencia/11-armazenamento-de-arquivos.md) | URL pré-assinada, upload que **não passa pelo backend**, LocalStack — e o que a assinatura não garante |

### 03 — Testes e integração

Como provar que funciona: contratos, erros, banco real, segurança e carga.

| # | Documento | O que você aprende |
|---|---|---|
| 01 | [Stubs e contract tests](./03-testes-integracao/01-stubs-contract-tests.md) | Testar integração entre serviços sem subir todos: Spring Cloud Contract, WireMock, Stub Runner |
| 02 | [Tratamento de erros na API](./03-testes-integracao/02-tratamento-erros-api.md) | `ProblemDetail` (RFC 9457), hierarquia de exceções e quando usar 404, 422 ou 500 |
| 03 | [Testes de integração de query services](./03-testes-integracao/03-testes-integracao-query-services.md) | Query service se testa contra banco real: Testcontainers, isolamento do seed do Flyway, builders e os cenários mínimos de paginação |
| 04 | [Testes de carga com k6](./03-testes-integracao/04-testes-de-carga-k6.md) | Cenários, executores e thresholds — e as três armadilhas que fazem um teste de carga medir menos do que promete |
| 05 | [Testando segurança](./03-testes-integracao/05-testando-seguranca.md) | Identidade em teste com `@WithSecurityContext`, declarativo × imperativo, o que o `MockJwtDecoder` **não** cobre — e por que teste de asserção negativa dá falso conforto |
| 06 | [Cartões de teste](./03-testes-integracao/06-cartoes-testes.md) | Consulta rápida: os cartões do FastPay cadastrados no billing e o resultado fixo de cada um no checkout |

### 04 — Infraestrutura

Como tudo roda: ambiente local, containers, cache, saúde, descoberta e gateways.

| # | Documento | O que você aprende |
|---|---|---|
| 01 | [Ambiente local](./04-infraestrutura/01-ambiente-local.md) | Do clone aos serviços rodando: submódulos, Docker Compose, portas, bancos e problemas comuns |
| 02 | [Docker](./04-infraestrutura/02-docker.md) | Build de imagem, multi-arquitetura com Buildx e publicação em registry |
| 03 | [Carga de dados no MongoDB](./04-infraestrutura/03-carga-de-dados-mongo.md) | `ApplicationRunner`, Extended JSON e por que isso **não** substitui o Flyway |
| 04 | [Jobs agendados](./04-infraestrutura/04-scheduled-jobs.md) | `@Scheduled`, execução em ambiente distribuído e controle de concorrência |
| 05 | [Redis na prática](./04-infraestrutura/05-redis.md) | Cache sem persistência, política de eviction, inspeção — e a interpolação do Compose que deixou tudo sem senha |
| 06 | [Resiliência na prática](./04-infraestrutura/06-resiliencia-config.md) | Os parâmetros por cliente, a biblioteca que quase ninguém documenta, e como testar retry por contagem de requests |
| 07 | [Health check e degradação](./04-infraestrutura/07-health-checks.md) | Liveness × readiness, um status `DEGRADED` inventado — e um indicador que funciona ao lado de outro que mente |
| 08 | [Threads e concorrência](./04-infraestrutura/08-threads-e-concorrencia.md) | Onde o sistema realmente satura — e por que ligar threads virtuais o deixou **9× pior** |
| 09 | [Service discovery](./04-infraestrutura/09-service-discovery.md) | O endereço que saiu da configuração: Eureka Server standalone, clients que se anunciam, os dois builders do load balancer — e o jeito novo de falhar que o discovery traz |
| 10 | [API Gateway](./04-infraestrutura/10-api-gateway.md) | A porta de entrada virou uma só: rotas à mão por service ID (e por que a ordem delas é regra de negócio), CORS global, token na borda — e a rota que o `denyAll` matou antes do roteamento |
| 11 | [Resiliência na borda](./04-infraestrutura/11-resiliencia-no-gateway.md) | Os cinco padrões chegam ao gateway com OUTRA biblioteca (Resilience4j reactor): retry que muda a unidade da janela do breaker, cache sobre cache, e rate limit por usuário que falha aberto — de propósito |
| 12 | [BFF e gateways por cliente](./04-infraestrutura/12-bff-e-gateways-por-cliente.md) | A borda dividida por público: cache por necessidade, JSON do tamanho do cliente, API composition com degradação por ramo — e por que BFF não é sinônimo de gateway |

### 05 — Segurança

Quem pode chamar o quê: OAuth 2.1, OIDC, PKCE, papéis e segredos.

| # | Documento | O que você aprende |
|---|---|---|
| 01 | [Identidade e fundamentos do OAuth 2](./05-seguranca/01-fundamentos-identidade-oauth2.md) | Senha × certificado × token, IdP e IAM, os quatro papéis, grants e por que escopo é **teto** e não papel |
| 02 | [Authorization Server](./05-seguranca/02-authorization-server.md) | Os dois clientes do projeto, token **opaco × JWT**, e quem guarda as chaves |
| 03 | [Resource servers e escopos](./05-seguranca/03-resource-server-e-escopos.md) | Quem **exige** o token: escopo por rota, 401 × 403 e a matriz de 142 testes |
| 04 | [OAuth2 client e token](./05-seguranca/04-oauth2-client-e-token.md) | Quem **pede** o token: as três peças, o cache por principal, e a linha de config que acoplava a subida de um serviço à do outro |
| 05 | [Authorization code e consentimento](./05-seguranca/05-authorization-code-e-consentimento.md) | O fluxo **com pessoa**: código, `state`, consentimento granular, refresh com rotação — e a propriedade no singular que desligava a rotação em silêncio |
| 06 | [Telas e formulários de login](./05-seguranca/06-telas-e-formularios-de-login.md) | O contrato invisível entre o HTML e o filtro, o `_csrf` que o Thymeleaf injeta, consentimento próprio — e por que um teste de login pode passar com a tela quebrada |
| 07 | [OpenID Connect: identidade, sessão e logout](./05-seguranca/07-openid-connect-e-sessao.md) | ID token × access token, usuário no banco, `/userinfo`, logout com revogação — e por que ele **não** alcança um JWT já emitido |
| 08 | [Gestão de usuários e auditoria](./05-seguranca/08-gestao-de-usuarios-e-auditoria.md) | A API de usuários, `/me` como id que o cliente não escolhe, token de pessoa × de máquina — e a auditoria que trocou o `UUID` aleatório pelo `sub` do token |
| 09 | [Verificação de e-mail e troca de senha](./05-seguranca/09-verificacao-de-email-e-troca-de-senha.md) | Token com hash no banco, o agregado orquestrando a regra, ativação e recuperação como o **mesmo** fluxo — e por que não se deve dizer quem tem conta |
| 10 | [PKCE e clientes públicos](./05-seguranca/10-pkce-e-clientes-publicos.md) | Como um cliente **sem segredo** prova continuidade; silent refresh por `prompt=none`; cookie de domínio comum, CORS × `frame-ancestors` |
| 11 | [RBAC e controle de acesso](./05-seguranca/11-rbac-e-controle-de-acesso.md) | O fluxo guiado das quatro camadas: papel no token, client e escopo por papel, regras de negócio e dono do recurso — e a lacuna entre duas tabelas que quebrou a loja |
| 12 | [Recursos `/me` e IDOR](./05-seguranca/12-recursos-me-e-idor.md) | O id que sai do path e do body nos quatro serviços: `/me` com filtro de dono **na consulta**, os três públicos das anotações (CUSTOMER, interno, máquina) — e a SpEL quebrada que respondia 500 |
| 13 | [Segredos centralizados e a chave RSA](./05-seguranca/13-segredos-centralizados-e-chave-rsa.md) | Parameter Store × Secrets Manager, o seed por CSV que aposentou os comandos manuais, `spring.config.import` como dependência de bootstrap — e a chave de assinatura que parou de mudar a cada subida (quase) |

### 06 — Mensageria

Como os serviços conversam por eventos e coordenam processos longos. Os docs desta pasta são **didáticos**: cada um traz o problema, a ideia, o fluxo no AlgaShop, como aplicar e o que lembrar. O detalhe de revisão fica no diário de bordo.

| # | Documento | O que você aprende |
|---|---|---|
| 01 | [Fundamentos de EDA](./06-mensageria/01-fundamentos-eda.md) | O que é um evento, os tipos de evento, fila × log e coreografia × orquestração — o mapa antes do broker |
| 02 | [Kafka a fundo](./06-mensageria/02-kafka-fundamentos.md) | Tópico, partição, key, offset e consumer group: como o Kafka garante ordem e paralelismo |
| 03 | [Kafka na prática](./06-mensageria/03-kafka-na-pratica.md) | O primeiro evento ponta a ponta: o catálogo publica, o ordering consome, e o `__TypeId__` desacopla os pacotes |
| 04 | [ECST e validação de eventos](./06-mensageria/04-ecst-e-validacao-de-eventos.md) | O evento que carrega o estado: o consumidor reage sem chamar o produtor, e valida o que recebe |
| 05 | [Contratos com AsyncAPI](./06-mensageria/05-contratos-asyncapi.md) | Escrever o contrato da mensageria: canais, operações e mensagens de cada serviço |
| 06 | [Resiliência no producer](./06-mensageria/06-resiliencia-no-producer.md) | O que o produtor consegue garantir: acks, idempotência e o que fazer quando o envio falha |
| 07 | [Resiliência no consumer](./06-mensageria/07-resiliencia-no-consumer.md) | Quando o consumo falha: retry com backoff, erro que não vale retentar e a DLT |
| 08 | [Outbox](./06-mensageria/08-outbox.md) | Gravar no banco e publicar no Kafka sem perder nada: a mensagem nasce na mesma transação |
| 09 | [Workflow coreografado](./06-mensageria/09-workflow-coreografado.md) | Um processo que atravessa serviços só por eventos: pedido → fatura → pagamento → pedido pago |
| 10 | [Saga coreografada](./06-mensageria/10-saga-coreografada.md) | Desfazer o que já foi feito em outro serviço: compensações por eventos, reserva e estorno |
| 11 | [Saga orquestrada — conceito e mapa](./06-mensageria/11-saga-orquestrada-conceito-e-mapa.md) | Um coordenador que sabe em que passo cada pedido está: o fluxo inteiro, feliz e com falhas |
| 12 | [Saga orquestrada — preparando o terreno](./06-mensageria/12-saga-orquestrada-preparando-o-terreno.md) | Por que o checkout assíncrono e a saga coreografada saíram antes de orquestrar |
| 13 | [Saga orquestrada — estado e persistência](./06-mensageria/13-saga-orquestrada-estado-e-persistencia.md) | A `PlaceOrderSaga`: status × passo, falhas, e como a saga é guardada e começa |
| 14 | [Saga orquestrada — comandos e respostas](./06-mensageria/14-saga-orquestrada-comandos-e-respostas.md) | Comando, resposta e correlação: reply manual × `@SendTo`, até o caminho feliz completo |
| 15 | [Saga orquestrada — compensações](./06-mensageria/15-saga-orquestrada-compensacoes.md) | Pagamento recusado e falta de estoque: como o orquestrador desfaz os passos |
| 16 | [Saga orquestrada — robustez](./06-mensageria/16-saga-orquestrada-robustez.md) | Validar antes de começar, comandos sem outbox e validação de payload no consumidor |
| 99 | [Diário de bordo da mensageria](./06-mensageria/99-diario-de-bordo.md) | Revisões, bugs encontrados, pendências e testes de cada etapa — consulta, não leitura em sequência |

### 07 — Notificações

O serviço que entrega e-mail aos clientes. Está em construção: os docs crescem a cada aula, e as pendências ficam no próprio doc.

| # | Documento | O que você aprende |
|---|---|---|
| 01 | [Serviço de notificações: o conceito](./07-notificacoes/01-servico-de-notificacoes-conceito.md) | *Quando* × *como* notificar, por que o e-mail fica fora da saga, `SendEmail` pela outbox e idempotência por id derivado do evento |
| 02 | [Validação do HTML do template](./07-notificacoes/02-validacao-de-html-do-template.md) | Validar na entrada com jsoup: erros de parse que o parser esconderia, lista de bloqueio, ofuscação de `javascript:` e por que os `th:*` também entram |
| 03 | [Renderização do template com Thymeleaf](./07-notificacoes/03-renderizacao-do-template-com-thymeleaf.md) | Template do banco com `StringTemplateResolver`, assunto em TEXT × corpo em HTML, escape automático — e o render a seco que reprova sintaxe sem exigir valores |
| 04 | [Envio de e-mail por SMTP](./07-notificacoes/04-envio-de-email-por-smtp.md) | Porta `EmailSender` × adapter JavaMail, `spring.mail` × remetente validado na subida, falha de montagem × de envio — e o Mailpit como SMTP falso |
| 05 | [API de cadastro de templates](./07-notificacoes/05-api-de-templates.md) | CRUD que valida antes de gravar, 404 antes de 422, UUIDv7 gerado na aplicação e o `version` nulo que decide persist × merge, `PagedModel` — com o roteiro de testes do Postman |
| 06 | [API de envio de e-mails](./07-notificacoes/06-api-de-envio-de-emails.md) | Corpo como **comando** (o mesmo tipo vai vir pelo Kafka), `@Valid` aninhado, destinatário que as variáveis não sobrescrevem, 204/400/404/422/502 — e log sem dado pessoal |
| 07 | [Segurança: resource server](./07-notificacoes/07-seguranca-resource-server.md) | As duas pontas (auth server emite, notifications exige), secret com `{bcrypt}` no Parameter Store, escopo por meta-anotação, 401 × 403 sem corpo — e o `$` a mais que quebrava o hash |

### Artefatos

| Pasta | Conteúdo |
|---|---|
| [`domain-diagram/`](./domain-diagram/) | Diagramas de domínio em PDF — ordering, billing, product-catalog |
| [`openapi/`](./openapi/) | Especificações OpenAPI dos três serviços com API REST |

---

## Consulta rápida por assunto

| Preciso lembrar de… | Vá para |
|---|---|
| Diferença entre agregado e entidade | [Arquitetura](./00-visao-geral/01-arquitetura.md) |
| Onde colocar uma regra de negócio nova | [Ports & Adapters](./01-arquitetura-design/01-ports-hexagonal.md), [Specification](./01-arquitetura-design/03-specification.md) |
| Por que a listagem não devolve o objeto completo | [CQRS](./01-arquitetura-design/02-cqrs.md), [MongoDB](./02-persistencia/04-product-catalog-mongo.md) |
| Embutir ou referenciar no Mongo | [Normalizado × desnormalizado](./02-persistencia/08-desnormalizacao-mongo.md), [MongoDB](./02-persistencia/04-product-catalog-mongo.md) |
| Montar um filtro com N parâmetros opcionais | [Consultas com Criteria](./02-persistencia/05-consultas-mongo-criteria.md) |
| Comparar dois campos do mesmo documento | [Consultas com Criteria](./02-persistencia/05-consultas-mongo-criteria.md) |
| Juntar duas coleções no Mongo | [Aggregation Pipeline](./02-persistencia/07-agregacoes-mongo.md) |
| Resolver N+1 no Mongo | [Normalizado × desnormalizado](./02-persistencia/08-desnormalizacao-mongo.md), [Aggregation Pipeline](./02-persistencia/07-agregacoes-mongo.md) |
| Manter uma cópia desnormalizada em dia | [Eventos e listeners](./01-arquitetura-design/04-eventos-e-listeners.md) |
| Publicar um evento de domínio a partir do agregado | [Eventos e listeners](./01-arquitetura-design/04-eventos-e-listeners.md) |
| Evitar que duas escritas concorrentes se atropelem | [Concorrência e atomicidade](./02-persistencia/09-concorrencia-e-atomicidade.md) |
| Evitar ir ao banco toda vez | [Cache](./01-arquitetura-design/05-cache.md) |
| Invalidar cache sem servir dado velho | [Cache](./01-arquitetura-design/05-cache.md) |
| Deixar o navegador reaproveitar a resposta (`ETag`, `304`) | [Cache](./01-arquitetura-design/05-cache.md) |
| Configurar TTL, eviction e senha do Redis | [Redis na prática](./04-infraestrutura/05-redis.md) |
| Impedir que um serviço lento derrube o meu | [Resiliência](./01-arquitetura-design/06-resiliencia.md) |
| Decidir o que vale retentar (e o que não vale) | [Resiliência](./01-arquitetura-design/06-resiliencia.md) |
| Saber quando um fallback é honesto | [Resiliência](./01-arquitetura-design/06-resiliencia.md) |
| Escrever um teste que prova retry e circuito | [Resiliência na prática](./04-infraestrutura/06-resiliencia-config.md) |
| Saber se o serviço pode receber tráfego | [Health check e degradação](./04-infraestrutura/07-health-checks.md) |
| Diferença entre liveness e readiness | [Health check e degradação](./04-infraestrutura/07-health-checks.md) |
| Dependência opcional fora sem derrubar o serviço | [Health check e degradação](./04-infraestrutura/07-health-checks.md) |
| Saber quanto o sistema aguenta | [Testes de carga com k6](./03-testes-integracao/04-testes-de-carga-k6.md) |
| Escolher entre VUs fixos e taxa fixa no k6 | [Testes de carga com k6](./03-testes-integracao/04-testes-de-carga-k6.md) |
| Fazer um teste de carga reprovar de verdade | [Testes de carga com k6](./03-testes-integracao/04-testes-de-carga-k6.md) |
| Decidir se vale ligar threads virtuais | [Threads e concorrência](./04-infraestrutura/08-threads-e-concorrencia.md) |
| Descobrir qual recurso satura primeiro | [Threads e concorrência](./04-infraestrutura/08-threads-e-concorrencia.md) |
| Entender por que o container morreu com exit 137 | [Threads e concorrência](./04-infraestrutura/08-threads-e-concorrencia.md), [Docker](./04-infraestrutura/02-docker.md) |
| Receber upload de arquivo sem passar pelo backend | [Armazenamento de arquivos](./02-persistencia/11-armazenamento-de-arquivos.md) |
| Saber o que uma URL pré-assinada garante (e o que não) | [Armazenamento de arquivos](./02-persistencia/11-armazenamento-de-arquivos.md) |
| Emular a AWS localmente | [Armazenamento de arquivos](./02-persistencia/11-armazenamento-de-arquivos.md), [Ambiente local](./04-infraestrutura/01-ambiente-local.md) |
| Ter dois adapters para a mesma porta | [Ports & Adapters](./01-arquitetura-design/01-ports-hexagonal.md) |
| Diferença entre autenticar e autorizar | [Identidade e OAuth 2](./05-seguranca/01-fundamentos-identidade-oauth2.md) |
| Escolher entre token opaco e JWT | [Authorization Server](./05-seguranca/02-authorization-server.md) |
| Entender por que um JWT não se revoga | [Authorization Server](./05-seguranca/02-authorization-server.md) |
| Saber o que é escopo (e o que ele **não** é) | [Identidade e OAuth 2](./05-seguranca/01-fundamentos-identidade-oauth2.md) |
| Escolher o grant type certo | [Identidade e OAuth 2](./05-seguranca/01-fundamentos-identidade-oauth2.md) |
| Descobrir para que serve o `/oauth2/jwks` | [Authorization Server](./05-seguranca/02-authorization-server.md) |
| Proteger uma rota por escopo | [Resource servers e escopos](./05-seguranca/03-resource-server-e-escopos.md) |
| Diferença entre 401 e 403 | [Resource servers e escopos](./05-seguranca/03-resource-server-e-escopos.md), [Tratamento de erros](./03-testes-integracao/02-tratamento-erros-api.md) |
| Saber se desligar CSRF é seguro | [Resource servers e escopos](./05-seguranca/03-resource-server-e-escopos.md) |
| Testar autorização sem subir o authorization server | [Resource servers e escopos](./05-seguranca/03-resource-server-e-escopos.md) |
| Chamar outro serviço protegido a partir do meu | [OAuth2 client e token](./05-seguranca/04-oauth2-client-e-token.md) |
| Obter e cachear token entre serviços | [OAuth2 client e token](./05-seguranca/04-oauth2-client-e-token.md) |
| Evitar que um serviço dependa do outro **para subir** | [OAuth2 client e token](./05-seguranca/04-oauth2-client-e-token.md) |
| Autenticar uma **pessoa** e pedir consentimento | [Authorization code e consentimento](./05-seguranca/05-authorization-code-e-consentimento.md) |
| Entender por que existe um *código* antes do token | [Authorization code e consentimento](./05-seguranca/05-authorization-code-e-consentimento.md) |
| Fazer refresh token rotacionar de verdade | [Authorization code e consentimento](./05-seguranca/05-authorization-code-e-consentimento.md) |
| Saber o que o authorization server precisa persistir | [Authorization code e consentimento](./05-seguranca/05-authorization-code-e-consentimento.md) |
| Diferença entre ID token e access token | [OpenID Connect e sessão](./05-seguranca/07-openid-connect-e-sessao.md) |
| Autenticar usuário contra o banco | [OpenID Connect e sessão](./05-seguranca/07-openid-connect-e-sessao.md) |
| Migrar de algoritmo de senha sem invalidar as antigas | [OpenID Connect e sessão](./05-seguranca/07-openid-connect-e-sessao.md) |
| Fazer logout revogar tokens de verdade | [OpenID Connect e sessão](./05-seguranca/07-openid-connect-e-sessao.md) |
| Guardar sessão fora da memória | [OpenID Connect e sessão](./05-seguranca/07-openid-connect-e-sessao.md) |
| Saber **quem** está chamando, sem acoplar a aplicação ao Spring Security | [Gestão de usuários e auditoria](./05-seguranca/08-gestao-de-usuarios-e-auditoria.md) |
| Preencher `createdBy` com o usuário de verdade | [Gestão de usuários e auditoria](./05-seguranca/08-gestao-de-usuarios-e-auditoria.md) |
| Distinguir token de pessoa de token de máquina | [Gestão de usuários e auditoria](./05-seguranca/08-gestao-de-usuarios-e-auditoria.md) |
| Fazer um endpoint `/me` sem receber id na URL | [Recursos `/me` e IDOR](./05-seguranca/12-recursos-me-e-idor.md), [Gestão de usuários e auditoria](./05-seguranca/08-gestao-de-usuarios-e-auditoria.md) |
| Apagar um usuário sem quebrar referência de auditoria | [Gestão de usuários e auditoria](./05-seguranca/08-gestao-de-usuarios-e-auditoria.md) |
| Autenticar uma SPA ou app mobile sem `client_secret` | [PKCE e clientes públicos](./05-seguranca/10-pkce-e-clientes-publicos.md) |
| Entender o que o PKCE protege (e o que não protege) | [PKCE e clientes públicos](./05-seguranca/10-pkce-e-clientes-publicos.md) |
| Renovar token no navegador sem refresh token | [PKCE e clientes públicos](./05-seguranca/10-pkce-e-clientes-publicos.md) |
| Compartilhar sessão entre subdomínios | [PKCE e clientes públicos](./05-seguranca/10-pkce-e-clientes-publicos.md) |
| Diferença entre CORS e `frame-ancestors` | [PKCE e clientes públicos](./05-seguranca/10-pkce-e-clientes-publicos.md) |
| Diferença entre escopo e papel (e por que os dois) | [RBAC e controle de acesso](./05-seguranca/11-rbac-e-controle-de-acesso.md) |
| Transformar papel do usuário em permissão do Spring | [RBAC e controle de acesso](./05-seguranca/11-rbac-e-controle-de-acesso.md) |
| Impedir que um usuário use um aplicativo que não é dele | [RBAC e controle de acesso](./05-seguranca/11-rbac-e-controle-de-acesso.md) |
| Limitar escopos por papel, no mesmo client | [RBAC e controle de acesso](./05-seguranca/11-rbac-e-controle-de-acesso.md) |
| Fazer alguém ver só os próprios recursos | [RBAC e controle de acesso](./05-seguranca/11-rbac-e-controle-de-acesso.md) |
| Decidir entre filtrar a consulta e negar com 403 | [RBAC e controle de acesso](./05-seguranca/11-rbac-e-controle-de-acesso.md) |
| Eliminar IDOR em vez de mitigar (id fora do path e do body) | [Recursos `/me` e IDOR](./05-seguranca/12-recursos-me-e-idor.md) |
| Impedir que um recurso alheio confirme a própria existência | [Recursos `/me` e IDOR](./05-seguranca/12-recursos-me-e-idor.md) |
| Restringir um endpoint a token de máquina (ou excluir máquina dele) | [Recursos `/me` e IDOR](./05-seguranca/12-recursos-me-e-idor.md) |
| Preencher um campo do input pelo controller sem o body alcançá-lo | [Recursos `/me` e IDOR](./05-seguranca/12-recursos-me-e-idor.md) |
| Tirar segredos e configuração do YAML versionado | [Segredos centralizados e a chave RSA](./05-seguranca/13-segredos-centralizados-e-chave-rsa.md) |
| Decidir entre Parameter Store e Secrets Manager | [Segredos centralizados e a chave RSA](./05-seguranca/13-segredos-centralizados-e-chave-rsa.md) |
| Fixar a chave que assina os JWT (e entender o `kid`) | [Segredos centralizados e a chave RSA](./05-seguranca/13-segredos-centralizados-e-chave-rsa.md) |
| Semear a AWS local sem comando manual (CSV + init hook) | [Segredos centralizados e a chave RSA](./05-seguranca/13-segredos-centralizados-e-chave-rsa.md) |
| Chamar outro serviço pelo nome, sem URL fixa | [Service discovery](./04-infraestrutura/09-service-discovery.md) |
| Balancear chamadas entre réplicas de um serviço | [Service discovery](./04-infraestrutura/09-service-discovery.md) |
| Registrar um serviço no Eureka (e saber quem não deve) | [Service discovery](./04-infraestrutura/09-service-discovery.md) |
| Ter uma borda de entrada para cada front | [API Gateway](./04-infraestrutura/10-api-gateway.md), [BFF e gateways por cliente](./04-infraestrutura/12-bff-e-gateways-por-cliente.md) |
| Configurar CORS na borda certa (e saber quando nem precisa) | [BFF e gateways por cliente](./04-infraestrutura/12-bff-e-gateways-por-cliente.md) |
| Barrar requisição sem token antes de chegar no serviço | [API Gateway](./04-infraestrutura/10-api-gateway.md) |
| Limitar requisições por usuário (rate limit) | [Resiliência na borda](./04-infraestrutura/11-resiliencia-no-gateway.md) |
| Cachear resposta na borda (e somar as camadas de cache) | [Resiliência na borda](./04-infraestrutura/11-resiliencia-no-gateway.md) |
| Impedir que um serviço lento derrube o gateway | [Resiliência na borda](./04-infraestrutura/11-resiliencia-no-gateway.md) |
| Montar um BFF (e entender por que ele dispensa PKCE) | [BFF e gateways por cliente](./04-infraestrutura/12-bff-e-gateways-por-cliente.md) |
| Compor respostas de vários endpoints numa só | [BFF e gateways por cliente](./04-infraestrutura/12-bff-e-gateways-por-cliente.md) |
| Otimizar o JSON por tipo de cliente | [BFF e gateways por cliente](./04-infraestrutura/12-bff-e-gateways-por-cliente.md) |
| Entender EDA antes de escolher broker | [Fundamentos de EDA](./06-mensageria/01-fundamentos-eda.md) |
| Decidir entre fila e log (RabbitMQ × Kafka) | [Fundamentos de EDA](./06-mensageria/01-fundamentos-eda.md) |
| Escolher entre coreografia e orquestração | [Fundamentos de EDA](./06-mensageria/01-fundamentos-eda.md) |
| Entender partições, offsets e consumer groups | [Kafka a fundo](./06-mensageria/02-kafka-fundamentos.md) |
| Decidir a key (e o número de partições) de um tópico | [Kafka a fundo](./06-mensageria/02-kafka-fundamentos.md) |
| Publicar e consumir o primeiro evento no Kafka | [Kafka na prática](./06-mensageria/03-kafka-na-pratica.md) |
| Desacoplar pacotes de produtor e consumidor (`__TypeId__`) | [Kafka na prática](./06-mensageria/03-kafka-na-pratica.md) |
| Aplicar ECST e reagir a eventos no consumidor | [ECST e validação de eventos](./06-mensageria/04-ecst-e-validacao-de-eventos.md) |
| Validar eventos com Bean Validation (produtor e consumidor) | [ECST e validação de eventos](./06-mensageria/04-ecst-e-validacao-de-eventos.md) |
| Documentar contratos de eventos (AsyncAPI) | [AsyncAPI: contratos](./06-mensageria/05-contratos-asyncapi.md) |
| Garantir entrega no producer (acks, idempotence, rollback) | [Resiliência no producer](./06-mensageria/06-resiliencia-no-producer.md) |
| Tratar falha no consumo (retry, backoff, DLT) | [Resiliência no consumer](./06-mensageria/07-resiliencia-no-consumer.md) |
| Deduplicar mensagens (idempotência de negócio) | [Resiliência no consumer](./06-mensageria/07-resiliencia-no-consumer.md) |
| Implementar outbox (e entender o checkout 202 que existiu até a Fase 43) | [Outbox](./06-mensageria/08-outbox.md) |
| Encadear serviços num processo por eventos (coreografia) | [Workflow coreografado](./06-mensageria/09-workflow-coreografado.md) |
| Desfazer passos de outros serviços quando algo falha (saga) | [Saga coreografada](./06-mensageria/10-saga-coreografada.md) |
| Coordenar uma saga com um orquestrador | [Saga orquestrada — conceito e mapa](./06-mensageria/11-saga-orquestrada-conceito-e-mapa.md) |
| Modelar o estado de uma saga (status × passo) | [Saga orquestrada — estado e persistência](./06-mensageria/13-saga-orquestrada-estado-e-persistencia.md) |
| Mandar um comando e receber a resposta (correlação, `@SendTo`) | [Saga orquestrada — comandos e respostas](./06-mensageria/14-saga-orquestrada-comandos-e-respostas.md) |
| Desfazer passos quando um serviço recusa (compensação orquestrada) | [Saga orquestrada — compensações](./06-mensageria/15-saga-orquestrada-compensacoes.md) |
| Validar o payload que chega pelo Kafka | [Saga orquestrada — robustez](./06-mensageria/16-saga-orquestrada-robustez.md) |
| Separar quem decide *quando* notificar de quem entrega o e-mail | [Serviço de notificações: o conceito](./07-notificacoes/01-servico-de-notificacoes-conceito.md) |
| Evitar e-mail duplicado com entrega pelo menos uma vez | [Serviço de notificações: o conceito](./07-notificacoes/01-servico-de-notificacoes-conceito.md) |
| Validar o HTML de um template antes de salvar | [Validação do HTML do template](./07-notificacoes/02-validacao-de-html-do-template.md) |
| Renderizar um template guardado no banco (Thymeleaf sem MVC) | [Renderização do template com Thymeleaf](./07-notificacoes/03-renderizacao-do-template-com-thymeleaf.md) |
| Impedir que uma variável vire HTML no e-mail (escape) | [Renderização do template com Thymeleaf](./07-notificacoes/03-renderizacao-do-template-com-thymeleaf.md) |
| Enviar e-mail HTML por SMTP com JavaMail | [Envio de e-mail por SMTP](./07-notificacoes/04-envio-de-email-por-smtp.md) |
| Testar e-mail sem mandar de verdade (Mailpit) | [Envio de e-mail por SMTP](./07-notificacoes/04-envio-de-email-por-smtp.md) |
| Gerar id UUIDv7 na aplicação sem quebrar o `save` do Spring Data | [API de cadastro de templates](./07-notificacoes/05-api-de-templates.md) |
| Devolver página no formato `content` + `page` (`PagedModel`) | [API de cadastro de templates](./07-notificacoes/05-api-de-templates.md) |
| Validar campos de um objeto aninhado no corpo (`@Valid`) | [API de envio de e-mails](./07-notificacoes/06-api-de-envio-de-emails.md) |
| Impedir que o cliente troque o destinatário pelas variáveis | [API de envio de e-mails](./07-notificacoes/06-api-de-envio-de-emails.md) |
| Proteger um serviço novo com JWT e escopo por endpoint | [Segurança: resource server](./07-notificacoes/07-seguranca-resource-server.md) |
| Cadastrar um cliente `client_credentials` com secret no Parameter Store | [Segurança: resource server](./07-notificacoes/07-seguranca-resource-server.md) |
| Rever um bug, uma pendência ou os testes de uma etapa da mensageria | [Diário de bordo](./06-mensageria/99-diario-de-bordo.md) |
| Trocar a tela de login padrão do Spring Security | [Telas e formulários de login](./05-seguranca/06-telas-e-formularios-de-login.md) |
| Descobrir por que o login devolve 403 (ou nunca funciona) | [Telas e formulários de login](./05-seguranca/06-telas-e-formularios-de-login.md) |
| Fazer uma tela de consentimento própria | [Telas e formulários de login](./05-seguranca/06-telas-e-formularios-de-login.md) |
| Servir CSS e imagem numa aplicação protegida | [Telas e formulários de login](./05-seguranca/06-telas-e-formularios-de-login.md) |
| Dar baixa em estoque sem vender o que não tem | [Concorrência e atomicidade](./02-persistencia/09-concorrencia-e-atomicidade.md) |
| Fazer duas escritas caírem juntas, ou nenhuma | [Transações e replica set](./02-persistencia/10-transacoes-mongo.md) |
| Por que `@Transactional` no Mongo pode não fazer nada | [Transações e replica set](./02-persistencia/10-transacoes-mongo.md) |
| Subir um Mongo descartável no teste (Testcontainers) | [Transações e replica set](./02-persistencia/10-transacoes-mongo.md), [Ambiente local](./04-infraestrutura/01-ambiente-local.md) |
| Calcular campo derivado no banco em vez de em Java | [Aggregation Pipeline](./02-persistencia/07-agregacoes-mongo.md) |
| Saber se minha consulta usa índice | [Índices no MongoDB](./02-persistencia/06-indices-mongo.md) |
| Por que meu índice existe e não foi usado | [Índices no MongoDB](./02-persistencia/06-indices-mongo.md) |
| Ordenar resultado de busca por relevância | [Índices no MongoDB](./02-persistencia/06-indices-mongo.md) |
| Popular o Mongo com dados de teste | [Carga de dados](./04-infraestrutura/03-carga-de-dados-mongo.md) |
| Autenticar um usuário dentro de um teste | [Testando segurança](./03-testes-integracao/05-testando-seguranca.md) |
| Saber se meu teste de segurança prova alguma coisa | [Testando segurança](./03-testes-integracao/05-testando-seguranca.md) |
| Descobrir o que um `JwtDecoder` mockado deixa de cobrir | [Testando segurança](./03-testes-integracao/05-testando-seguranca.md) |
| Substituir uma dependência que nasce dentro de um `@Bean` | [Testando segurança](./03-testes-integracao/05-testando-seguranca.md) |
| Devolver 404 ou 422 | [Tratamento de erros](./03-testes-integracao/02-tratamento-erros-api.md) |
| Criar uma migration | [Flyway](./02-persistencia/01-flyway.md) |
| Testar sem subir o outro serviço | [Contract tests](./03-testes-integracao/01-stubs-contract-tests.md) |
| Separar teste rápido de teste que precisa de banco | [Contract tests](./03-testes-integracao/01-stubs-contract-tests.md) |
| Em que porta roda cada coisa | [Ambiente local](./04-infraestrutura/01-ambiente-local.md) |
| Comandos de submódulo Git | [Ambiente local](./04-infraestrutura/01-ambiente-local.md) |

---

## Como começar a rodar

```bash
git clone --recurse-submodules https://github.com/gabriel-lima258/algashop-meta.git
cd algashop-meta

docker compose -f docker-compose.tools.yml up -d

cd microservices/algashop-ordering
./gradlew bootRun
```

Passo a passo completo, mapa de portas e solução de problemas: **[Ambiente local](./04-infraestrutura/01-ambiente-local.md)**.

---

## Stack

**Java 25** · **Spring Boot 4.0** · Spring Data JPA · Spring Data MongoDB · PostgreSQL 17 · MongoDB 8 em replica set · Redis 8 · Flyway · Gradle 9 · Spring Cloud Contract · Spring Cloud CircuitBreaker · Spring Boot Actuator · WireMock · Testcontainers · JUnit 5 · AssertJ · ModelMapper · Lombok · Docker Compose

---

## Sobre a organização deste repositório

As pastas **e os arquivos** são numerados para dar **ordem de leitura**, não só agrupamento — a numeração vai do mais conceitual (visão geral, design) ao mais operacional (infraestrutura), e termina na mensageria, que junta quase tudo o que veio antes.

Os documentos das pastas 00 a 05 seguem a estrutura:

1. **O problema** que o conceito resolve
2. **A solução** explicada com código real do projeto
3. **Armadilhas** encontradas na prática
4. **Pendências registradas** — o que ficou por fazer, declarado abertamente
5. **Checklist** de revisão

Os da mensageria (06) seguem um modelo mais didático — **Em uma frase · O problema · A ideia · Como funciona no AlgaShop · Como aplicar · Para lembrar · Cuidados · Onde ver no código** — e as revisões, bugs e pendências de cada etapa ficam no [diário de bordo](./06-mensageria/99-diario-de-bordo.md).

As pendências são propositais: documentação que só mostra o que deu certo não ajuda a estudar. Elas marcam onde o próximo aprendizado começa.
