# Docker

Guia de build e execução das imagens Docker dos microsserviços do Algashop.

## Pré-requisitos

- Docker 20+ instalado e em execução
- JAR do serviço já gerado em `build/libs/` (rode `./gradlew bootJar` antes do build da imagem)
- Para builds multi-arquitetura: Docker Buildx habilitado (incluso no Docker Desktop)

## Build simples (arquitetura local)

A partir do diretório do microsserviço (ex.: `microservices/algashop-ordering`):

```bash
docker build . -t usuario_docker_name/ordering:dev
```

- `.` — contexto de build (diretório atual)
- `-t algashop/ordering:dev` — nomeia a imagem como `algashop/ordering` com a tag `dev`

## Executando a imagem

```bash
docker run --rm -p 8080:8080 usuario_docker_name/ordering:dev
```

- `--rm` — remove o container ao encerrar
- `-p 8080:8080` — mapeia a porta do host para a porta do container

Para passar opções da JVM:

```bash
docker run --rm -p 8080:8080 -e JAVA_OPTS="-Xmx512m" algashop/ordering:dev
```

## Build multi-arquitetura (linux/amd64 + linux/arm64)

Útil para publicar imagens compatíveis com Macs Apple Silicon (arm64) e servidores x86 (amd64).

### 1. Criar um builder Buildx (apenas na primeira vez)

```bash
docker buildx create --name algashop-builder --use
docker buildx inspect --bootstrap
```

### 2. Build para múltiplas plataformas

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64/v8 \
  --tag usuario_docker_name/ordering:dev \
  --load \
  .
```

- `--platform` — lista de arquiteturas alvo separadas por vírgula
- `--load` — carrega a imagem no Docker local (só funciona para uma única plataforma por vez)
- `--push` — substitui `--load` quando quiser publicar direto em um registry

### 3. Build e push para um registry

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64/v8 \
  --tag usuario_docker_name/ordering:dev \
  --push \
  .
```

## A cadeia de `include`

São três arquivos, em dois saltos:

```
docker-compose.yml  ->  docker-compose.services.yml  ->  docker-compose.tools.yml
```

`docker compose up -d`, sem argumento nenhum, sobe tudo. Incluir os dois no arquivo raiz seria erro — o Compose recusa serviço declarado duas vezes.

```bash
docker compose up -d                                  # tudo
docker compose -f docker-compose.tools.yml up -d      # só a infraestrutura
```

## Limite de memória não é detalhe de arrumação

```yaml
deploy:
  resources:
    limits:
      memory: 512M
```

Parece configuração cosmética até o dia em que o container **desaparece**:

```
ExitCode=137  OOMKilled=true
```

Foi o que aconteceu com o `ordering` a 256M, por volta de **1600 req/s** — medido, não estimado. Não houve stack trace nem log de erro da aplicação: o kernel matou o processo e o cliente passou a receber conexão recusada.

Para conferir a causa de um container que sumiu:

```bash
docker inspect <container> --format '{{.State.ExitCode}} {{.State.OOMKilled}}'
```

O `product-catalog` já rodava em 512M pelo mesmo motivo, sem que ninguém tivesse registrado o porquê. Agora está registrado — e vale a regra geral: **um limite que aperta antes esconde todos os outros.** Enquanto a memória estourava, era impossível medir qualquer coisa sobre threads. Ver [`threads-e-concorrencia.md`](./threads-e-concorrencia.md).

## Um container que só existe para configurar outros

Nem todo serviço do compose precisa continuar de pé. O `algashop-mongodb-init` sobe, roda um comando e morre — é o que transforma três `mongod` isolados num replica set:

```yaml
algashop-mongodb-init:
  image: mongo:8
  depends_on:
    algashop-mongodb-1: { condition: service_healthy }
    algashop-mongodb-2: { condition: service_healthy }
    algashop-mongodb-3: { condition: service_healthy }
  entrypoint: ["bash", "-c"]
  command: >
    "mongosh --host algashop-mongodb-1 --eval 'rs.initiate({...})' || true;"
  restart: "no"
```

Três detalhes carregam o peso todo:

- **`condition: service_healthy`** — sem isso o `rs.initiate` correria contra um `mongod` ainda subindo. `depends_on` sozinho só espera o container **iniciar**, não ficar pronto; é o `healthcheck` que dá sentido à condição.
- **`|| true`** — `rs.initiate` num conjunto já iniciado retorna erro. Sem isso, todo `docker compose up` a partir do segundo terminaria em falha.
- **`restart: "no"`** — este container **deve** terminar. Sem a diretiva, a política padrão o reiniciaria em loop depois de cada saída bem-sucedida.

O padrão vale além do Mongo: sempre que a configuração de um serviço precisa acontecer *de fora* dele, um container efêmero com `depends_on` + `healthcheck` é mais simples que um script de entrypoint em cada nó — e fica visível no `compose ps` como um passo que rodou.

Detalhes de por que o replica set existe: [`../02-persistencia/transacoes-mongo.md`](../02-persistencia/transacoes-mongo.md).

## ⚠️ A imagem base e o toolchain andam juntos

O `Dockerfile` e o `build.gradle` guardam a mesma informação em dois lugares, e nada os amarra:

```dockerfile
FROM eclipse-temurin:25-jre      # Dockerfile
```
```gradle
languageVersion = JavaLanguageVersion.of(25)   // build.gradle
```

Quando o toolchain sobe e a imagem não, o build continua **verde** — o Gradle compila normalmente. A falha só aparece no `docker run`:

```
Exception in thread "main" java.lang.UnsupportedClassVersionError:
  ... has been compiled by a more recent version of the Java Runtime
  (class file version 69.0), this version of the Java Runtime only
  recognizes class file versions up to 65.0
```

A tradução do número: **69 = Java 25**, **65 = Java 21**. É a soma `44 + versão`, e vale decorar — ela aparece em toda migração de JDK.

Foi exatamente o que aconteceu aqui: os quatro serviços migraram para Java 25 e os quatro `Dockerfile` continuaram em `21-jre`. O build passava, a suíte passava, e nenhuma imagem subia.

Para conferir contra o que foi realmente compilado, sem depender do que o `build.gradle` promete:

```bash
javap -v build/classes/java/main/.../Application.class | grep major
```

## Dicas e problemas comuns

- **Tag da base `eclipse-temurin`**: use `eclipse-temurin:25-jre` (com hífen). A variante `25.jre` não existe no Docker Hub. E a versão precisa acompanhar o toolchain — ver a seção acima.
- **JAR não encontrado**: rode `./gradlew clean bootJar` antes do `docker build` — o Dockerfile espera o artefato em `build/libs/`.
- **Limpar builders**: `docker buildx rm algashop-builder` remove o builder criado.
- **O `product-catalog` só ganhou Dockerfile na Fase 17** — era o único dos quatro sem imagem. Junto vieram o `bootJar { archiveFileName = 'product-catalog.jar' }` e a task `dockerBuild`, e o serviço entrou no `docker-compose.services.yml`.
- **O `JAR_NAME` do Dockerfile tem que casar com o `archiveFileName` do `build.gradle`.** O `ADD` copia `build/libs/$JAR_NAME`; divergir quebra o build da imagem com um erro que não diz isso.
- **As imagens são publicadas em `gabriel58221/*`**, e o compose passou a apontar para lá. Antes ele referenciava `algashop/*`, que nenhuma task publicava — `docker compose up` puxava (ou não achava) uma imagem que ninguém construía.

> ⚠️ **Nenhum dos quatro Dockerfiles tem `HEALTHCHECK`**, mesmo depois de os serviços passarem a expor `/actuator/health/readiness`. O Docker considera o container saudável assim que o processo sobe. Fechar isso exigiria `curl` ou `wget` na imagem — a base `eclipse-temurin:25-jre` não traz nenhum dos dois — ou um bloco `healthcheck:` no compose. Ver [`health-checks.md`](./health-checks.md).
- **Container de init aparece como `Exited (0)`**: é o esperado. `docker compose logs algashop-mongodb-init` mostra o resultado do `rs.initiate`. O mesmo vale para o `algashop-billing-scheduler`, que entrou no compose na Fase 18: ele é um job, roda uma vez e encerra.

---

## O authorization server entrou no compose (Fase 26)

Ele era o único serviço que só rodava por `bootRun`. Ganhou `Dockerfile` — idêntico ao dos outros — e a mesma task `dockerBuild` com Buildx multi-plataforma.

Uma armadilha que custou um `docker compose up` inteiro: o nome da imagem no `build.gradle` e o nome no `docker-compose.services.yml` **precisam ser o mesmo**. Estavam diferentes (`gabriel58221/...` contra `algashop/...`), e o sintoma não diz isso:

```
Error response from daemon: pull access denied for algashop/authorization-server,
repository does not exist or may require 'docker login'
```

O Docker não tem como saber que a imagem *deveria* ter sido construída localmente: se o nome não está na máquina, ele tenta puxar do registry, e a mensagem fala de autenticação — não de nome errado.

> **Imagem local e imagem do compose são amarradas só pela string.** Nada valida a correspondência: é a mesma família do nome de bean em SpEL e do escopo em `hasAuthority`.
