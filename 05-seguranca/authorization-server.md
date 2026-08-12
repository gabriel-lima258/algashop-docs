# O Authorization Server do AlgaShop

> O sexto repositório do projeto, e o que menos tem código: uma dependência, uma classe vazia e dois clientes em YAML. Este documento é sobre o que essa configuração decide — em especial a escolha entre **token opaco e JWT**, que é a que muda mais coisa.
> Código real: `microservices/authorization-server/` · Contrato: [`openapi/authorization-server.yml`](../openapi/authorization-server.yml)
> Conceitos e vocabulário em [Identidade e fundamentos do OAuth 2](./fundamentos-identidade-oauth2.md).

> 🔄 **Atualizado na Fase 21.** Os dois clientes trocaram `reference` por `self-contained` — o contraste opaco × JWT descrito abaixo deixou de existir na prática, e a seção correspondente explica por quê. A configuração de quem valida está em [Resource servers e escopos](./resource-server-e-escopos.md).

> ⚠️ **Este documento foi escrito sem executar o servidor.** Toda resposta e todo token mostrados aqui são **ilustrativos** — derivados da especificação e do contrato OpenAPI, e assinalados como tal. Onde a afirmação dependeria de rodar, o texto diz isso. É o oposto das fases 17, 18 e 19, e a diferença está explícita de propósito.

---

## O serviço inteiro

```
authorization-server/
├── build.gradle                                     ← uma dependência
├── src/main/java/.../AuthorizationServerApplication.java   ← @SpringBootApplication vazia
└── src/main/resources/
    ├── application.yaml                             ← grupos de perfil
    ├── application-base.yaml                        ← porta e nome
    ├── application-development-env.yaml             ← OS DOIS CLIENTES
    ├── application-docker-env.yaml                  ← vazio
    └── application-production-env.yaml              ← vazio
```

```gradle
implementation 'org.springframework.boot:spring-boot-starter-security-oauth2-authorization-server'
```

Uma linha de dependência e uma classe sem corpo produzem **seis endpoints funcionando**: `/oauth2/authorize`, `/oauth2/token`, `/oauth2/introspect`, `/oauth2/revoke`, `/oauth2/jwks` e `/.well-known/oauth-authorization-server`.

> É o que vale entender do Spring Authorization Server: ele **não** é uma biblioteca para você implementar OAuth. Ele já implementa o protocolo inteiro. O que resta configurar é **quem pode pedir token e o que esse token vale** — nunca *como* o protocolo funciona.

**Porta 9000.** Era 8081, a mesma do `algashop-ordering`, e os dois não subiam juntos. 9000 é a convenção do Spring e já era a porta usada nos exemplos do contrato.

**O `issuer` é explícito** desde a Fase 21:

```yaml
spring.security.oauth2.authorizationserver.issuer: http://algashop-authorization-server:9000
```

Ele vai dentro do claim `iss` de todo token emitido, e é o mesmo valor que os resource servers configuram como `issuer-uri` — quem valida **compara**. Fixar dos dois lados é o que impede um token de outro emissor de passar. E é por isso que o nome precisa resolver na máquina de quem desenvolve, não só dentro da rede do compose: há uma linha para ele em `etc/hostnames/hostnames`.

---

## Anatomia de um cliente registrado

O arquivo da fase é `application-development-env.yaml`. O Spring lê tudo abaixo de `spring.security.oauth2.authorizationserver.client` e monta um `RegisteredClientRepository` **em memória**.

```yaml
spring:
  security:
    oauth2:
      authorizationserver:
        client:
          algashop-test-client:            # ← chave do mapa, não é o client-id
            registration:
              client-id: algashop-test
              authorization-grant-types:
                - client_credentials
              client-authentication-methods:
                - client_secret_basic
              client_secret: "{noop}testing123"
              scopes:
                - products:read
                - products:write
            token:
              access-token-time-to-live: 15m
              access-token-format: reference        # ← opaco
```

Campo a campo:

| Campo | O que decide |
|---|---|
| a chave do mapa | só um apelido interno para agrupar a configuração — **não** é o `client-id` |
| `client-id` | o identificador público, o "usuário" do cliente |
| `client-authentication-methods` | **como** ele prova quem é. `client_secret_basic` = `client-id:secret` no header `Authorization: Basic` |
| `client_secret` | o segredo. `{noop}` é instrução ao `PasswordEncoder`: **texto puro, sem hash** |
| `authorization-grant-types` | quais procedimentos de obtenção de token ele pode usar |
| `scopes` | o **teto** do que qualquer token dele pode fazer |
| `access-token-time-to-live` | validade |
| `access-token-format` | `reference` (opaco) ou `self-contained` (JWT) |

### Três coisas que essa configuração diz nas entrelinhas

**O repositório é em memória.** Cadastrar cliente é editar YAML e reiniciar; quem sai do arquivo deixa de existir na próxima subida. Não há tabela nem API de administração. Para estudo é o certo — a alternativa (`JdbcRegisteredClientRepository`) traria schema, migration e um CRUD antes de qualquer conceito ter sido aprendido.

**`{noop}` é o `PasswordEncoder` sendo instruído a não fazer nada.** O prefixo entre chaves é como o Spring Security escolhe o algoritmo por credencial — `{bcrypt}`, `{argon2}`, `{noop}`. Em produção seria `{bcrypt}` e o valor viria de variável de ambiente ou de um cofre, **não** de um arquivo versionado.

**`client_secret` está em *snake_case*** enquanto as chaves vizinhas usam kebab-case. O *relaxed binding* do Boot aceita as duas formas, então funciona. É inconsistência, não defeito — mas é o tipo de coisa que faz o próximo leitor duvidar de qual é a certa.

---

## Os dois clientes

| | `algashop-test` | `algashop-ordering-service` |
|---|---|---|
| Para que existe | teste manual | o `ordering` chamando o catálogo |
| Grant | `client_credentials` | `client_credentials` |
| Escopos | **os 16** do sistema | **`products:read`** |
| TTL | 15 min | **5 min** |
| Formato | `self-contained` (JWT) | `self-contained` (JWT) |

> 🔄 **Isto mudou na Fase 21.** O cliente de teste era `reference` (opaco), e os dois juntos serviam para mostrar os dois caminhos. Quando os resource servers entraram, foram configurados com `.jwt()` — que só sabe validar token auto-contido. Token opaco exigiria que cada serviço chamasse `/oauth2/introspect`, e nenhum deles faz isso.
>
> A leitura vale mais que a mudança: **a escolha do formato do token não é do emissor, é do sistema.** Quem valida define o que consegue validar, e o emissor se ajusta. O contraste descrito abaixo continua sendo o raciocínio certo — ele só deixou de estar exercitado aqui.

O cliente de teste carrega os 16 escopos de todos os serviços porque é ele que faz `curl` na mão. O do `ordering` carrega **um**, porque o `ordering` só lê o catálogo — e **o escopo mais estreito que faz o trabalho é o certo**. Um token dele que vaze não escreve nada.

---

## Opaco × JWT: a escolha que decide o resto

É a decisão mais consequente do arquivo, e ela se resume a uma pergunta: **o resource server precisa perguntar a alguém para validar o token?**

### Token opaco (`reference`)

Uma string sem conteúdo. Não há nada a decodificar — é uma **referência** que só o emissor sabe resolver.

```
Authorization: Bearer 7Zk2Nq...          (ilustrativo)
```

Para saber se vale e o que autoriza, o resource server chama `/oauth2/introspect`:

```json
// resposta ilustrativa, conforme RFC 7662 e o schema do contrato
{ "active": true, "scope": "products:read products:write",
  "client_id": "algashop-test", "exp": 1735689600, "sub": "algashop-test" }
```

### JWT (`self-contained`)

Três partes em Base64URL separadas por ponto — header, payload, assinatura. Os claims viajam **dentro** do token:

```
eyJhbGciOiJSUzI1NiIsImtpZCI6IjEifQ.eyJzdWIiOiJhbGdhc2hvcC1vcmRlcmluZy1zZXJ2aWNlIiwi...
└──────────── header ────────────┘ └──────────── payload ───────────...
```

```json
// payload ilustrativo
{ "iss": "http://localhost:9000", "sub": "algashop-ordering-service",
  "aud": "algashop-ordering-service", "scope": ["products:read"],
  "exp": 1735689600, "iat": 1735689300, "jti": "..." }
```

O resource server valida **localmente**, conferindo a assinatura com a chave pública. Não chama ninguém.

### O que isso custa e o que isso compra

| | `reference` (opaco) | `self-contained` (JWT) |
|---|---|---|
| O portador carrega | uma referência sem conteúdo | os claims, assinados |
| Validação | chamada a `/oauth2/introspect` | verificação de assinatura, local |
| Custo por requisição | **uma ida à rede** | nenhuma |
| Se o authorization server cair | ninguém valida nada | **tudo continua funcionando** |
| Revogar | **imediato** | só quando expirar |
| Estado no emissor | precisa guardar o token | não precisa |
| Vaza informação? | nada | **tudo que está nos claims** |

> ⚠️ **JWT é assinado, não cifrado.** Qualquer um que tenha o token lê o payload — Base64 não é criptografia. Assinatura garante que ninguém *alterou* os claims, não que ninguém os *leu*. Nunca coloque num JWT o que você não colocaria num cartão postal.

### Por que 15 minutos e 5 minutos

Os TTLs parecem arbitrários e não são. Eles são consequência direta do formato:

> **Com JWT, o tempo de vida é a janela de exposição de um token vazado** — não existe como cancelá-lo. Quem tem o token tem o acesso até o `exp`, e o emissor não é consultado para descobrir o contrário.
>
> Com token opaco há revogação de verdade: um `POST /oauth2/revoke` e a próxima introspecção devolve `active: false`. O TTL pode ser mais generoso porque ele deixou de ser a única defesa.

Daí 5 minutos para o JWT e 15 para o opaco. É a mesma lógica que aparece em [resiliência](../01-arquitetura-design/resiliencia.md): quando um mecanismo de recuperação não existe, o parâmetro que limita o dano tem que compensar.

### Como escolher

| Escolha | Quando |
|---|---|
| **JWT** | muitos resource servers, alta vazão, tolerância a revogação lenta, e independência do emissor |
| **Opaco** | revogação precisa ser imediata, o token pode chegar a terceiros, ou o volume não justifica abrir mão do controle |

O híbrido comum em produção: **JWT internamente, opaco para fora**. Serviços internos validam local e barato; um token que sai da sua fronteira é opaco, porque você não controla quem vai lê-lo.

---

## Quem tem as chaves

É o que faz o JWT funcionar, e a mesma propriedade do certificado descrita nos [fundamentos](./fundamentos-identidade-oauth2.md#três-coisas-que-provam-identidade): **quem valida nunca precisa do segredo**.

```
AUTHORIZATION SERVER          RESOURCE SERVER
  chave privada                  busca /oauth2/jwks
  assina o JWT   ───token───▶    confere a assinatura
                                 com a chave PÚBLICA
```

O `/oauth2/jwks` publica as chaves públicas em JSON:

```json
// ilustrativo, conforme RFC 7517 e o schema do contrato
{ "keys": [ { "kty": "RSA", "use": "sig", "kid": "1", "alg": "RS256",
              "n": "...", "e": "AQAB" } ] }
```

**Por que um endpoint e não copiar a chave em cada serviço.** Por causa do `kid`. Rotacionar chave exige um período em que a antiga e a nova coexistem — tokens já emitidos precisam continuar válidos. O `kid` no header do JWT diz *qual* chave assinou; o resource server busca o conjunto e escolhe. Com a chave copiada em arquivo, rotação vira deploy coordenado de todos os serviços ao mesmo tempo.

> ⚠️ **Sem configuração de chave, o Spring gera um par novo a cada subida.** É o que acontece hoje aqui: nada no repositório define uma chave persistente. **Reiniciar o servidor invalida todo JWT já emitido** — em desenvolvimento é invisível (o TTL é de 5 minutos), em produção é uma interrupção. Não verificado por execução, mas é o comportamento padrão documentado do projeto.

E o assunto é maior que uma pendência: a chave privada de assinatura é a credencial mais valiosa do sistema inteiro. Quem a tiver **emite tokens válidos para qualquer coisa** — não precisa quebrar nenhum serviço, só assinar o que quiser.

---

## Os endpoints

Lidos do contrato ([`openapi/authorization-server.yml`](../openapi/authorization-server.yml)):

| Endpoint | Quem chama | Para quê |
|---|---|---|
| `POST /oauth2/token` | o **client** | trocar credenciais (ou código) por token |
| `POST /oauth2/introspect` | o **resource server** | resolver um token opaco |
| `POST /oauth2/revoke` | o **client** | cancelar um token antes da hora |
| `GET /oauth2/jwks` | o **resource server** | pegar as chaves públicas |
| `GET /.well-known/oauth-authorization-server` | qualquer um | descobrir todos os outros |
| `GET /oauth2/authorize` | o **navegador** | iniciar o fluxo com usuário |

Os três primeiros exigem autenticação do cliente; os três últimos são públicos — e é correto que sejam: chave pública é pública, e metadado de descoberta existe para ser descoberto.

### O `.well-known` é mais importante do que parece

Ele devolve `issuer`, `token_endpoint`, `jwks_uri`, `grant_types_supported`, `scopes_supported`. Um resource server configurado apenas com o **issuer** descobre o resto sozinho — inclusive onde buscar as chaves.

> É por isso que configurar resource server em Spring costuma ser uma linha: `spring.security.oauth2.resourceserver.jwt.issuer-uri`. O resto vem daqui.

### `/oauth2/authorize` está documentado e **não é utilizável hoje**

O contrato descreve o endpoint completo — `code_challenge`, `nonce`, `prompt`, `max_age`. Mas:

- nenhum dos dois clientes tem `authorization_code` nos `authorization-grant-types`
- não há `redirect-uri` configurada
- não há **nenhum usuário** cadastrado para autenticar

Ou seja: o contrato documenta **o protocolo**, não a capacidade atual deste servidor. É ótimo como referência de estudo e enganoso como contrato — quem ler o YAML sem ler o YAML de configuração vai supor um fluxo que não existe.

---

## Como testar (quando o servidor subir)

> Estes comandos **não foram executados** nesta fase. Estão aqui para serem rodados.

```bash
cd microservices/authorization-server && ./gradlew bootRun
```

**Token opaco** — o cliente de teste:

```bash
curl -s -u algashop-test:testing123 \
  -d grant_type=client_credentials \
  -d scope="products:read products:write" \
  http://localhost:9000/oauth2/token
```

**Token JWT** — o cliente do `ordering`:

```bash
curl -s -u algashop-ordering-service:secret123 \
  -d grant_type=client_credentials \
  -d scope=products:read \
  http://localhost:9000/oauth2/token
```

A diferença deve saltar aos olhos: o primeiro `access_token` é uma string sem estrutura; o segundo tem dois pontos e decodifica.

**Resolver o token opaco:**

```bash
curl -s -u algashop-test:testing123 \
  -d token=<ACCESS_TOKEN> \
  http://localhost:9000/oauth2/introspect
```

**As chaves e os metadados:**

```bash
curl -s http://localhost:9000/oauth2/jwks
curl -s http://localhost:9000/.well-known/oauth-authorization-server
```

**As provas negativas**, que valem mais que as positivas:

```bash
# segredo errado                    -> 401 invalid_client
curl -s -u algashop-test:errado -d grant_type=client_credentials \
  http://localhost:9000/oauth2/token

# escopo que o cliente não tem      -> 400 invalid_scope
curl -s -u algashop-ordering-service:secret123 \
  -d grant_type=client_credentials -d scope=products:write \
  http://localhost:9000/oauth2/token
```

A segunda é a que demonstra a frase "escopo é teto": o cliente **pede** `products:write` e é recusado, porque escrita não está no registro dele.

---

## O que isto ainda não protege

**Nada.** E é importante dizer com essa clareza.

Emitir token não protege recurso nenhum enquanto **ninguém exigir o token**. Hoje `ordering` e `billing` não têm uma linha de configuração de resource server: aceitam qualquer requisição, com ou sem `Authorization`. O ciclo só fecha quando eles recusarem.

### O catálogo já ganhou a dependência — e ela sozinha quebrou a suíte

No `product-catalog` o `build.gradle` recebeu:

```gradle
implementation 'org.springframework.boot:spring-boot-starter-security-oauth2-resource-server'
```

Sem nenhuma configuração acompanhando. E o efeito é imediato: `./gradlew check` no catálogo passou de **56 verdes** para **4 falhas**, todas com a mesma cara:

```
java.lang.AssertionError: Status expected:<200> but was:<401>
java.lang.AssertionError: Status expected:<400> but was:<401>
```

> **A lição vale mais que o incidente: acrescentar o starter de segurança ao classpath já tranca a aplicação inteira.** O Spring Boot ativa a autoconfiguração de segurança pela presença da dependência, e o padrão dela é `anyRequest().authenticated()` — todo endpoint passa a exigir autenticação, sem que ninguém tenha escrito uma linha pedindo isso.
>
> É o oposto do padrão que quase todo starter segue. `spring-boot-starter-data-redis` no classpath sem configuração não faz nada; `spring-boot-starter-security` sem configuração faz **tudo**. E é deliberado: falhar fechado é a escolha certa para segurança — o acidente que ele evita (subir desprotegido por esquecimento) é muito pior que o que ele causa (401 inesperado em teste).

Fechar isso exige as duas metades que ainda faltam: um `SecurityFilterChain` dizendo o que é público e o que exige escopo, e um `spring.security.oauth2.resourceserver.jwt.issuer-uri` apontando para `http://localhost:9000` — de onde o catálogo descobre sozinho o `jwks_uri` pelo `.well-known`.

A ordem, porém, está certa. Emitir vem antes de verificar, porque não dá para verificar o que não existe — e é o mesmo padrão de todo o projeto: [o contrato antes da implementação](../03-testes-integracao/stubs-contract-tests.md), o domínio antes do banco, o instrumento antes da medição.

---

## Armadilhas

- **JWT não é cifrado.** É legível por qualquer um que o tenha.
- **JWT não é revogável.** O TTL é a única defesa.
- **Chave efêmera.** Sem configuração, cada reinício invalida os tokens em circulação.
- **Segredo em texto puro, versionado.** `{noop}` no repositório é didático e indefensável fora de estudo.
- **Cliente em memória.** Cadastro é editar YAML e reiniciar.
- **`production` sobe sem cliente algum.** O grupo é `base + production-env`, e os clientes estão em `development-env`.
- **O contrato descreve mais do que o servidor faz.** `/oauth2/authorize` está documentado sem grant que o habilite.

---

## Pendências registradas

- [x] ~~**Nenhum resource server configurado.**~~ Resolvido na Fase 21: os três serviços validam token e cada rota exige um escopo, com 142 testes travando a matriz. Ver [Resource servers e escopos](./resource-server-e-escopos.md). Continua faltando a outra ponta — o `ordering` **não propaga token** ao chamar o catálogo, e o 401 resultante chega ao usuário como 422.
- [ ] **Chave de assinatura não persistida** — reiniciar invalida todo JWT emitido.
- [ ] **Segredos em texto puro no repositório.** Mínimo aceitável: `{bcrypt}` + variável de ambiente.
- [ ] **`production-env` e `docker-env` vazios** — sem cliente, sem issuer, sem chave.
- [ ] **Não está no `docker-compose`.** É o segundo serviço fora do compose depois do `billing-scheduler`, e este aqui *é* um serviço que fica de pé.
- [ ] **Sem Actuator**, ao contrário dos outros quatro. Não há `/actuator/health` para dizer se ele está pronto.
- [ ] **Sem teste além do `contextLoads`.** Um teste que peça token com os dois clientes e afirme o formato de cada um caberia em poucas linhas e travaria a configuração.
- [ ] **Sem usuário e sem `authorization_code`** — o fluxo com pessoa ainda não existe.
- [ ] **A auditoria continua gravando `UUID` aleatório** como autor no `product-catalog`. É a pendência mais antiga do projeto, e agora ela tem para onde ir: o `sub` do token.
- [ ] **Nada foi executado nesta fase.** Todos os exemplos vêm da especificação. Rodar os comandos acima e substituir por saída real é o próximo passo óbvio.

---

## Checklist de revisão

- [ ] O formato do token foi escolhido a partir da necessidade de revogação, e não por hábito?
- [ ] O TTL do JWT é curto o bastante para substituir a revogação que ele não tem?
- [ ] A chave de assinatura é persistente e rotacionável?
- [ ] Os segredos dos clientes estão fora do repositório e com hash?
- [ ] Cada cliente tem o **menor** conjunto de escopos que resolve o caso dele?
- [ ] Existe algum resource server efetivamente exigindo o token?
- [ ] O que está no payload do JWT poderia ser lido em público sem prejuízo?

---

## Referências

- [Spring Authorization Server — Reference](https://docs.spring.io/spring-authorization-server/reference/index.html)
- [RFC 7662 — Token Introspection](https://datatracker.ietf.org/doc/html/rfc7662)
- [RFC 7009 — Token Revocation](https://datatracker.ietf.org/doc/html/rfc7009)
- [RFC 7517 — JSON Web Key](https://datatracker.ietf.org/doc/html/rfc7517)
- [RFC 8414 — Authorization Server Metadata](https://datatracker.ietf.org/doc/html/rfc8414)
- [Identidade e fundamentos do OAuth 2](./fundamentos-identidade-oauth2.md)
- [`openapi/authorization-server.yml`](../openapi/authorization-server.yml) — o contrato dos endpoints
