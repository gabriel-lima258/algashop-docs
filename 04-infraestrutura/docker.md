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

## Dicas e problemas comuns

- **Tag da base `eclipse-temurin`**: use `eclipse-temurin:21-jre` (com hífen). A variante `21.jre` não existe no Docker Hub.
- **JAR não encontrado**: rode `./gradlew clean bootJar` antes do `docker build` — o Dockerfile espera o artefato em `build/libs/`.
- **Limpar builders**: `docker buildx rm algashop-builder` remove o builder criado.
