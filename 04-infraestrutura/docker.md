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
- **Container de init aparece como `Exited (0)`**: é o esperado. `docker compose logs algashop-mongodb-init` mostra o resultado do `rs.initiate`.
