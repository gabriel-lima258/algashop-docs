# Segredos centralizados: o cofre, o Parameter Store e a chave que parou de mudar

> Duas pendências antigas do caderno apontavam para o mesmo lugar: `client-secret: secret123` num arquivo versionado ([OAuth2 client e token](./oauth2-client-e-token.md)) e a chave de assinatura JWT que nascia nova a cada subida ([Authorization Server](./authorization-server.md)). Este módulo fecha as duas com a mesma resposta — a configuração sai do YAML e vai para a AWS emulada: Parameter Store para o que é config, Secrets Manager para o que é segredo, e um script de seed que aposentou os comandos manuais.
> Código real: `etc/aws/init.sh` + `parameters.csv`/`secrets.csv`/`s3.csv` · `spring.config.import` nos `application-*-env` dos 5 serviços · `infrastructure/security/jwk/JwkSourceConfig` e `JwkProperties` (authorization-server) · `docker-compose.tools.yml` (LocalStack).
> O emissor em [Authorization Server](./authorization-server.md) · quem consome token em [Resource servers e escopos](./resource-server-e-escopos.md) · o S3 que já morava no LocalStack em [Armazenamento de arquivos](../02-persistencia/armazenamento-de-arquivos.md).

---

## O problema tinha nome nos dois padrões

Até aqui, cada serviço carregava a própria configuração no próprio YAML — senha de banco, client secret, URL de integração, tudo junto e tudo versionado. Dois padrões nomeiam a saída:

**Configuração externalizada** (*externalized configuration*): o valor sai do artefato. O mesmo jar sobe em dev e em produção; o que muda é o que o ambiente injeta. O 12-Factor chama isso de "config no ambiente".

**Configuração centralizada** (*centralized configuration*): o valor sai de *cada máquina*. Em vez de cinco YAMLs repetindo a URL do authorization server, um lugar só guarda — e os cinco perguntam.

A AWS oferece os dois em serviços distintos, e a distinção é o desenho deste módulo:

| | Parameter Store (SSM) | Secrets Manager |
|---|---|---|
| Para quê | configuração: URLs, flags, portas | segredos: senhas, tokens, chaves |
| Forma do valor | escalar (`String`, `StringList`, `SecureString`) | JSON com múltiplas chaves |
| Namespace aqui | `/config/algashop/<serviço>/...` | `/secret/algashop/<serviço>/...` |

O prefixo do nome **é** a classificação. Quem lê `/secret/algashop/ordering/database` sabe que aquilo não aparece em log; quem lê `/config/algashop/ordering/redis/port` sabe que sim. E o JSON do Secrets Manager não é capricho:

```
/secret/algashop/ordering/database  →  {"username":"postgres","password":"postgres"}
```

Uma importação só entrega `ordering.secrets.datasource.username` **e** `.password` — o segredo é a *credencial inteira*, não cada campo dela.

---

## `spring.config.import` é dependência de bootstrap

Cada serviço declara de onde vem sua configuração — antes de qualquer bean existir:

```yaml
spring:
  config:
    import:
      - aws-secretsmanager:/secret/algashop/ordering/database?prefix=ordering.secrets.datasource.
      - aws-parameterstore:/config/algashop/ordering?prefix=ordering.params
      - aws-parameterstore:/config/algashop/shared?prefix=shared.params
```

E consome com placeholder comum:

```yaml
  datasource:
    url: ${ordering.params.datasource.url}
    username: ${ordering.secrets.datasource.username}
    password: ${ordering.secrets.datasource.password}
```

Três consequências que não estão escritas em lugar nenhum do YAML:

**1. O serviço não sobe sem o cofre.** O `import` roda na montagem do `Environment` — falha ali é falha antes do contexto, antes do datasource, antes de tudo. Por isso os cinco serviços do `docker-compose.services.yml` ganharam a mesma linha:

```yaml
      # config.import da AWS: sem o localstack pronto, o servico nem sobe.
      algashop-localstack:
        condition: service_healthy
```

Não é dependência de runtime como o Postgres — é dependência de *bootstrap*.

**2. O `prefix=` é o que evita colisão.** Cada import despeja suas propriedades num namespace próprio (`ordering.params.*`, `auth.secrets.*`), e o YAML referencia pelo nome completo. Sem isso, `url` do banco e `url` do gateway brigariam pelo mesmo nome.

**3. O tipo `StringList` não é decorativo.** `redirect-uris` e `allowedOrigins` são **listas** no YAML — e viraram um único `${auth.params.clients.algashop-ecommerce-web.redirect-uris}`. Funciona porque o binder do Spring converte a string separada por vírgula em `List<String>` na coerção. O tipo declarado no SSM documenta a intenção; o binder a executa.

E o achado de modelagem que paga o módulo: **`/config/algashop/shared/auth-server-url`**. A URL do issuer aparecia em cinco YAMLs — agora é um parâmetro, importado como `shared.params` por todo mundo que fala com o authorization server. A "fonte única do issuer", que antes era um comentário pedindo cuidado, virou infraestrutura.

---

## Do manual ao declarativo

A primeira versão deste módulo era um guia de comandos: `docker exec` no container, `awslocal secretsmanager create-secret` na mão, um por um. Funciona uma vez — e não sobrevive ao `docker compose down`, porque o LocalStack não persiste estado. A resposta é o hook de inicialização:

```yaml
    volumes:
      - ./etc/aws/init.sh:/etc/localstack/init/ready.d/init-aws.sh
      - ./etc/aws:/etc/aws
```

Tudo em `/etc/localstack/init/ready.d/` roda automaticamente quando os serviços do LocalStack ficam prontos. Não há passo manual no README — o seed *é parte da subida*. E o volume de dados existe porque grande volume não se digita: o `init.sh` lê três CSVs — **45 parâmetros**, **9 segredos** e **23 objetos S3**:

```bash
{
  read -r _header                       # descarta o cabeçalho
  while IFS=',' read -r name type value || [ -n "$name" ]; do
    [ -z "$name" ] && continue
    case "$name" in \#*) continue ;; esac
    value=${value%$'\r'}                # remove \r se foi salvo no Windows
    awslocal ssm put-parameter --name "$name" --type "$type" --value "$value" --overwrite
  done
} < /etc/aws/parameters.csv
```

Três linhas desse script escondem decisões que ninguém entende sem explicação:

**`awslocal configure set cli_follow_urlparam false`.** Por padrão, o AWS CLI ao receber um valor que começa com `http://` **baixa a URL e usa o corpo da resposta como valor**. Metade do `parameters.csv` são URLs. Sem essa linha, o seed gravaria HTML de erro no lugar de `jdbc:postgresql://...`.

**`IFS=','` com o valor como ÚLTIMO campo.** `read -r name type value` fatia a linha nas vírgulas — mas o último nome da lista recebe *todo o resto*, vírgulas incluídas. É o que faz o parser sobreviver a `StringList` de redirect-uris, à URI do Mongo com três hosts e ao JSON dos segredos. Parece frágil; é a semântica documentada do `read`.

**`value=${value%$'\r'}` e `|| [ -n "$name" ]`.** CSV salvo no Windows carrega `\r`; CSV sem quebra de linha final perde a última linha no `while read`. As duas guardas custam nada e evitam os dois bugs mais chatos de diagnosticar em seed de dados.

> **Comando manual é conhecimento que evapora; CSV com script é conhecimento que executa.** O guia de comandos foi absorvido por este documento e pelos arquivos em `etc/aws/` — o que sobrou dele são os comandos de *consulta*, adiante.

---

## O mesmo segredo, duas representações

O `parameters.csv` guarda os client secrets como `SecureString` — e com `{bcrypt}`:

```
/config/algashop/authorization-server/clients/algashop-ordering-service/secret,SecureString,{bcrypt}$2a$10$Bf4N...
```

E o `secrets.csv` guarda, para o ordering:

```
/secret/algashop/ordering/oauth2-client,{"client-secret":"secret123"}
```

Não é contradição — é o mesmo segredo visto dos dois lados do protocolo. O **authorization server** só precisa *verificar*: guarda o hash, e o `{noop}` didático de antes virou `{bcrypt}` de verdade. O **client** precisa *enviar*: precisa do texto, e o texto agora mora no cofre em vez de num arquivo versionado.

Duas pendências antigas fecham aqui: a de "`{noop}` em texto puro no repositório" do [Authorization Server](./authorization-server.md) e a de "`client-secret: secret123` versionado" do [OAuth2 client e token](./oauth2-client-e-token.md).

---

## A chave que parou de mudar

A pendência mais antiga da parte de segurança dizia: *reiniciar o authorization server invalida todo JWT emitido*. O detalhe notável é **por que** isso acontecia — não havia uma linha de código errada a apontar:

> **O comportamento indesejado vinha de não ter escrito nada.** Quando não existe um bean `JWKSource<SecurityContext>`, a autoconfiguração do Spring Authorization Server cria um par RSA novo, em memória, a cada subida. O default aparecia por ausência.

A correção tem duas metades. A primeira está no `init.sh` — a chave nasce no seed, não no repositório:

```bash
openssl genpkey -algorithm RSA -out /tmp/algashop-private-key.pem -pkeyopt rsa_keygen_bits:2048
PRIVATE_KEY_B64=$(base64 -w 0 /tmp/algashop-private-key.pem)
PRIVATE_KEY_ID=$(openssl rand -hex 16)

printf '{"privateKeyId":"%s","privateKey":"%s"}' "$PRIVATE_KEY_ID" "$PRIVATE_KEY_B64" > /tmp/secret.json

awslocal secretsmanager create-secret \
  --name /config/algashop/authorization-server/rsa-key \
  --secret-string file:///tmp/secret.json
```

Nenhum `.pem` versionado, nenhum keystore: a chave só existe no cofre e na memória do processo. O base64 (`-w 0`, linha única) é **transporte, não criptografia** — existe porque PEM tem quebras de linha e JSON de secret não gosta delas.

A segunda metade é o bean que a autoconfiguração passa a respeitar:

```java
@Bean
public JWKSource<SecurityContext> jwkSource(JwkProperties properties) ... {
    RSAPrivateKey rsaPrivateKey = RsaKeyConverters.pkcs8()
            .convert(new ByteArrayInputStream(privateKeyValue.getBytes(StandardCharsets.UTF_8)));

    RSAPrivateCrtKey privateCrtKey = (RSAPrivateCrtKey) rsaPrivateKey;
    RSAPublicKey rsaPublicKey = (RSAPublicKey) KeyFactory.getInstance("RSA")
            .generatePublic(new RSAPublicKeySpec(
                    privateCrtKey.getModulus(),
                    privateCrtKey.getPublicExponent()));   // ← a pública é DERIVADA, não guardada

    RSAKey rsaKey = new RSAKey.Builder(rsaPublicKey)
            .privateKey(rsaPrivateKey)
            .keyID(privateKeyId)                           // ← o kid do header de todo JWT
            .build();

    return new ImmutableJWKSet<>(new JWKSet(rsaKey));
}
```

O caminho do valor, ponta a ponta:

```
openssl genpkey → base64 → {"privateKeyId","privateKey"} → Secrets Manager
   → aws-secretsmanager:/config/.../rsa-key?prefix=jwk.
   → jwk.privateKey / jwk.privateKeyId
   → @ConfigurationProperties(prefix = "jwk")  JwkProperties
   → JwkSourceConfig → ImmutableJWKSet → /oauth2/jwks
```

Quatro decisões que valem ser lidas:

- **Só a privada é guardada.** Uma chave RSA no formato CRT carrega o módulo e o expoente público — `RSAPublicKeySpec(modulus, publicExponent)` reconstrói a pública quando quiser. Guardar as duas seria redundância, e é por isso que o `openssl rsa -pubout` do guia antigo não aparece no `init.sh`.
- **`RsaKeyConverters.pkcs8()` casa com `openssl genpkey`**, que produz PKCS#8 (`-----BEGIN PRIVATE KEY-----`). Trocar a geração por `openssl genrsa` (PKCS#1, `BEGIN RSA PRIVATE KEY`) quebra o conversor — armadilha registrada adiante.
- **O `keyID` é o `kid`** que vai no header de cada JWT e no `/oauth2/jwks` — é o que permite dois pares conviverem durante uma rotação. Ele nasce junto com a chave (`openssl rand -hex 16`).
- **`@Validated` + `@NotBlank`** nas `JwkProperties`: se o secret não chegou, o serviço falha na subida com mensagem de propriedade — não no primeiro token com um NPE.

### A pendência não morreu — mudou de dono

A chave agora sobrevive ao restart do authorization server. Mas o `openssl genpkey` roda **a cada recriação do container do LocalStack**, e o LocalStack não persiste estado: `docker compose down && up` → chave nova → todo JWT emitido antes vira inválido, exatamente o sintoma antigo com outro gatilho. Em desenvolvimento é aceitável (o TTL dos tokens é curto); o que importa é a forma da solução: **em produção o secret é criado uma vez, fora do ciclo de vida de qualquer container, e trocado por um procedimento de rotação — não por um restart.**

---

## Consultando o cofre (o que sobrou do guia manual)

Os comandos de leitura continuam úteis no dia a dia — agora com os paths reais:

```bash
# um segredo
awslocal secretsmanager get-secret-value --secret-id /secret/algashop/ordering/database

# um parâmetro
awslocal ssm get-parameter --name /config/algashop/ordering/datasource/url

# todos os parâmetros de um serviço, descriptografados
awslocal ssm get-parameters-by-path \
    --path /config/algashop/authorization-server --recursive --with-decryption

# a chave pública publicada (o que os resource servers consomem)
curl http://auth.algashop.local:9000/oauth2/jwks
```

Console gráfico: https://app.localstack.cloud (aponta para a instância local).

---

## Armadilhas

- **`/actuator/env` público com `show-values: always` republica o cofre em claro.** O `apiSecurityFilterChain` liberou `/actuator/**` e o perfil de dev expõe `env` com valores — senha de banco, client secrets e a chave privada (`jwk.privateKey`) saem por GET sem token. É decisão **deliberada e restrita ao `application-development-env.yaml`**: o `production-env` não pode herdar esse bloco `management`, e o `permitAll` de `/actuator/**` no código merece revisita antes de produção. Externalizar segredo não adianta se o processo o republica.
- **As credenciais AWS estão hardcoded no YAML** — a ironia do módulo que tirou segredos do YAML. São as chaves fake do LocalStack (`LSIAQ.../test`); em produção elas *somem*, junto com o `endpoint`: o SDK usa a cadeia default (IAM role da instância/pod), e é a ausência de credencial em arquivo que é o estado correto.
- **`create-secret` não é idempotente no script.** O `s3 mb` tem `|| true` com um comentário explicando por quê; os `create-secret` (da chave RSA e do loop de CSV) não têm `|| true` nem `--overwrite`. Só não quebra porque o LocalStack sobe vazio — com estado persistido, a segunda execução abortaria o seed no meio.
- **A chave RSA está no Secrets Manager com nome `/config/...`** — a única violação da convenção que os outros nove segredos seguem. O import compensa (`aws-secretsmanager:/config/...`), mas quem navegar pelo namespace vai classificá-la errado.
- **Config centralizada não elimina config por ambiente.** `mail/host = localhost` serve ao host; dentro do compose, `localhost` é o próprio container — e o perfil docker reafirma `spring.mail.host: algashop-mailpit`. O parâmetro central é o caso comum, não o único.
- **`base64 -w 0` é GNU coreutils** — roda no container do LocalStack; não roda no macOS de quem tentar executar o script na máquina.
- **PKCS#8, não PKCS#1** — ver acima; `genpkey` sim, `genrsa` não.
- **O parâmetro `/config/algashop/authorization-server/issuer` está órfão** — o issuer real vem de `shared.params.auth-server-url`. Parâmetro que ninguém lê é documentação que mente.

## Pendências registradas

- [ ] **A chave RSA continua efêmera — presa agora ao ciclo de vida do LocalStack.** Recriar o container gera chave nova e invalida os JWTs em circulação. Produção exige secret criado uma vez + procedimento de rotação (dois `kid` convivendo no JWKS durante a troca).
- [ ] **`production-env` segue vazio** nos cinco serviços — o perfil de produção não tem import, credencial nem endpoint definidos.
- [ ] **Restringir `/actuator/**` antes de produção** — hoje o `permitAll` cobre tudo e só o perfil decide o que existe.
- [ ] **Alinhar a rsa-key à convenção** (`/secret/...`) ou registrar a exceção no próprio nome.
- [ ] **Idempotência do seed** — `--overwrite`/`|| true` nos `create-secret`, pelo dia em que o LocalStack ganhar volume de estado.

## Checklist de revisão

- [ ] O valor é config ou segredo? `/config` (SSM) ou `/secret` (Secrets Manager) — o prefixo é a classificação.
- [ ] O segredo novo é JSON com todas as chaves da credencial, ou espalhou um campo por parâmetro?
- [ ] O serviço novo declarou `depends_on: algashop-localstack: service_healthy`?
- [ ] O `prefix=` do import é único no serviço?
- [ ] Valor com vírgula? Confira se ele é o último campo do CSV.
- [ ] Valor começa com `http`? Lembre do `cli_follow_urlparam`.
- [ ] Alguma chave/segredo voltou para YAML versionado no diff?
- [ ] O que o `/actuator/env` mostra neste perfil — e quem alcança o endpoint?

## Referências

- [Spring Cloud AWS — Secrets Manager e Parameter Store config import](https://docs.awspring.io/spring-cloud-aws/docs/3.0.0/reference/html/index.html)
- [AWS Secrets Manager](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html) · [AWS Systems Manager Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html)
- [LocalStack — Init hooks](https://docs.localstack.cloud/references/init-hooks/)
- [The Twelve-Factor App — Config](https://12factor.net/config)
- [Authorization Server](./authorization-server.md) · [OAuth2 client e token](./oauth2-client-e-token.md) · [Armazenamento de arquivos](../02-persistencia/armazenamento-de-arquivos.md) · [Ambiente local](../04-infraestrutura/ambiente-local.md)
