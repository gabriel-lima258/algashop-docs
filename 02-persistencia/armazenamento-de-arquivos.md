# Armazenamento de arquivos com URL pré-assinada

> Imagem de produto não é dado de domínio — é bytes. Este documento é sobre a decisão que organiza toda a feature: **nenhum byte passa pelo backend**. O cliente pede uma autorização, recebe uma URL assinada com prazo, e envia o arquivo direto para o S3.
> Código real: `application/storage/StorageProvider.java`, `infrastructure/storage/s3/`, `application/upload/`, `presentation/UploadRequestController.java` e `ProductImagesController.java` (product-catalog); LocalStack em `docker-compose.tools.yml` e `etc/aws/`.

---

## O problema

O reflexo natural é `@PostMapping(consumes = MULTIPART_FORM_DATA)` com um `MultipartFile`. Funciona na primeira demonstração e é caro de um jeito que só aparece sob carga.

Um upload de 5 MB pelo backend ocupa **uma thread do começo ao fim do tráfego** — não do processamento, do *tráfego*. Numa conexão móvel de 1 Mbps, são 40 segundos com uma thread parada segurando bytes no heap. A [Fase 18](../04-infraestrutura/threads-e-concorrencia.md) mediu este mesmo serviço com 10 threads no Tomcat: **dez uploads simultâneos e ninguém mais é atendido**.

E o backend não acrescenta nada no caminho. Ele recebe bytes e repassa bytes. É intermediário puro.

> A alternativa inverte quem faz o trabalho: o serviço **autoriza** e sai da frente. O tráfego acontece entre o cliente e o S3, que existe exatamente para isso.

---

## As duas fases

```
1. POST /api/v1/upload-requests           -> uploadSignedUrl + remoteFileName + expiresAt
2. PUT  <uploadSignedUrl>                 -> o CLIENTE envia os bytes ao S3
3. POST /api/v1/products/{id}/images      -> o produto passa a conhecer a imagem
   { "remoteFileName": "..." }
```

O que amarra as três é o **`remoteFileName`**. Ele nasce no passo 1, é a chave do objeto no passo 2, e volta ao serviço no passo 3.

Separar 1 e 3 tem uma consequência que precisa ser dita: entre eles **existe um intervalo em que o arquivo está no bucket e ninguém o reivindica**. Se o cliente desistir depois do passo 2, o objeto fica órfão — e nada hoje o recolhe.

### Passo 1 — autorizar

```java
FileReference fileReference = FileReference.builder()
        .contentLength(input.getContentLength())
        .contentType(mediaType)
        .fileName(UUID.randomUUID() + "." + extension)
        .expiresIn(Duration.ofMinutes(5))
        .allowPublicRead(true)
        .build();
```

**O nome enviado pelo cliente é descartado.** O que vai para o bucket é um UUID novo, e isso resolve três coisas de uma vez: colisão entre uploads, *path traversal* em nome malicioso, e vazamento de informação pelo nome original (`orcamento-cliente-x.jpg`).

O tipo é derivado da **extensão**, e só `jpeg` e `png` passam:

```java
MediaType mediaType = ImageMediaTypeExtractor.fromFileName(input.getOriginalFileName());
if (!(mediaType.equals(MediaType.IMAGE_JPEG) || mediaType.equals(MediaType.IMAGE_PNG))) {
    throw new IllegalArgumentException("Invalid image type");
}
```

> ⚠️ Extensão não é conteúdo. Um executável renomeado para `.png` passa por aqui sem qualquer objeção. Como o serviço nunca vê os bytes, **ele não tem como inspecionar o arquivo** — essa é uma troca real da arquitetura, não um descuido. Validar de verdade exigiria uma etapa assíncrona depois do upload (Lambda, evento do bucket, worker) lendo os *magic bytes*.

### Passo 2 — o cliente envia

Medido, com o serviço no ar:

```
PUT http://algashop-localstack:4566/algashop-product-image/4425a4f1-....jpg?X-Amz-Algorithm=...
  HTTP 200  enviados=24973 bytes

objeto no bucket:  24973  4425a4f1-9b50-452c-91f6-5aecd9145823.jpg
ocorrências do nome no log do serviço:  0
```

**Zero.** O catálogo não registrou nada porque não foi chamado. É a prova literal da tese.

### Passo 3 — reivindicar

```java
if (!storageProvider.fileExists(input.getRemoteFileName())) {
    throw new DomainException(...);
}
if (productRepository.existsByImagesName(input.getRemoteFileName())) {
    throw new DomainException(...);
}
```

Como o arquivo não passou por aqui, o catálogo **precisa perguntar** ao provedor se ele existe. Sem isso, um cliente que pediu a URL e nunca enviou nada grava no produto uma referência para imagem inexistente, e o erro só aparece no navegador de quem abrir a página.

Verificado:

```
POST .../images  { "remoteFileName": "nunca-subiu.jpg" }
  HTTP 422
  {"detail":"Image nunca-subiu.jpg was not found on storage provider", ...}
```

---

## O que a URL assinada garante — e o que não

A assinatura amarra **método, bucket, chave, content-type e prazo**. Nada além disso.

| Garante | Não garante |
|---|---|
| que só esta chave pode ser escrita | **o tamanho do arquivo** |
| que só até `expiresAt` | que o conteúdo seja mesmo uma imagem |
| que o `Content-Type` declarado é o enviado | que quem pediu seja quem envia |

O `contentLength` viaja pelo `UploadRequestInput`, é validado como positivo no `FileReference`… e **nunca é imposto**. Verificado:

```
autorização pedida para 100 bytes    -> HTTP 200
PUT com 391242 bytes                 -> HTTP 200
no bucket:  391242  b1984fc5-....png
```

**3.912 vezes o declarado, aceito sem reclamação.** PUT pré-assinado não limita bytes. Limitar de verdade exige `POST` com *policy* e `content-length-range` — outro formato de requisição, com campos de formulário em vez de corpo cru.

Vale entender por que isso importa mais aqui do que num upload comum: como o endpoint que emite autorizações **não tem autenticação** (o projeto inteiro não tem), qualquer um pede uma URL e escreve no bucket o quanto quiser. Num estudo local é irrelevante; numa conta AWS de verdade, é a fatura.

---

## A porta e os dois adapters

```java
// application/storage/StorageProvider.java
public interface StorageProvider {
    boolean healthCheck();
    URL requestUploadUrl(FileReference fileReference);
    void deleteFile(String remoteFileName);
    boolean fileExists(String remoteFileName);
}
```

A interface mora em `application/` e as implementações em `infrastructure/` — ela pertence a quem **precisa** dela, não a quem a satisfaz. Ver [Ports & Adapters](../01-arquitetura-design/ports-hexagonal.md).

Repare no que **não** existe na porta: nenhum método recebe ou devolve bytes. A arquitetura da feature está declarada na assinatura dos métodos.

São dois adapters, escolhidos por propriedade:

```java
@ConditionalOnProperty(name = "algashop.storage.provider", havingValue = "s3", matchIfMissing = true)
public class StorageProviderAwsS3Impl implements StorageProvider { ... }

@ConditionalOnProperty(name = "algashop.storage.provider", havingValue = "fake")
public class StorageProviderFakeImpl implements StorageProvider { ... }
```

> As condições **não são elegância — são requisito de inicialização**. São duas `@Component` implementando a mesma interface: sem elas o Spring não sabe qual injetar e a aplicação nem sobe. O `matchIfMissing = true` faz do S3 o padrão.

O fake é o que permite rodar o serviço e a suíte inteira **sem LocalStack de pé**. Ele devolve URL de mentira e responde `fileExists` com `true` — menos para `"fail.jpg"`, que existe justamente para exercitar a rejeição.

---

## O agregado

```java
private Image mainImage;
private Set<Image> images = new HashSet<>();
```

`mainImage` é uma **referência** para um dos elementos de `images`, não uma cópia. Duas invariantes protegem isso:

```java
public UUID addImage(String imageName) {
    Image image = new Image(imageName);
    this.images.add(image);
    if (this.mainImage == null) {
        this.setMainImage(image);   // a primeira imagem vira principal sozinha
    }
    return image.getId();
}

public void removeImage(UUID imageId) {
    Image image = findImageById(imageId);
    this.images.remove(image);
    if (image.equals(this.mainImage)) {
        this.setMainImage(this.images.stream().findFirst().orElse(null));
    }
}
```

Um produto **com** imagem nunca fica sem principal, e a principal nunca aponta para imagem que saiu da coleção. `null` só é legítimo quando não há nenhuma.

Verificado no ciclo completo — o produto já tinha imagem da carga, então a recém-anexada **não** foi promovida; depois de promovê-la e removê-la, outra voltou ao posto sozinha:

```
mainImage antes do delete:  4425a4f1-....jpg   (promovida por PUT .../primary)
DELETE .../images/{id}   -> 204, objeto removido do bucket
mainImage depois:           fe7787a7-....jpg   (promovida automaticamente)
```

**`equals` do `Image` é só por id.** Duas imagens com o mesmo nome de arquivo continuam sendo objetos diferentes — quem impede o mesmo arquivo de ser anexado duas vezes é o `existsByImagesName` do repositório. A regra "um arquivo pertence a um produto só" é do catálogo inteiro, não deste agregado, e por isso vive fora dele.

---

## A URL de leitura não é persistida

O documento guarda só o nome do arquivo. A URL é montada na saída, pelo ModelMapper:

```java
modelMapper.createTypeMap(Image.class, ImageOutput.class)
        .addMappings(m -> m.using(fromFileNameToUrlConverter)
                .map(Image::getName, ImageOutput::setUrl));
```

```yaml
algashop:
  mapping:
    image-storage-url: http://algashop-localstack:4566/algashop-product-image
```

É a escolha certa e vale saber por quê: **URL é endereço, não identidade**. Trocar de bucket, entrar num CDN ou mudar de região altera o endereço de toda imagem já existente — com a URL persistida, seria uma migração; com ela derivada, é uma linha de configuração.

`ApplicationMappingProperty` é `@Component` **incondicional** com `@Validated @NotBlank`: sem essa propriedade, nenhum ambiente sobe.

---

## LocalStack

```yaml
algashop-localstack:
  image: localstack/localstack:4.13.1
  ports: ["4566:4566", "4510-4559:4510-4559"]
  environment:
    SERVICES: s3
  volumes:
    - ./etc/aws/init.sh:/etc/localstack/init/ready.d/init-aws.sh
    - ./etc/aws:/etc/aws
    - ./etc/images:/etc/images
  deploy:
    resources:
      limits:
        memory: 1G
```

O gancho é o diretório: tudo em **`/etc/localstack/init/ready.d`** roda sozinho quando os serviços ficam prontos. Não há `docker exec` na mão nem passo manual no README.

```bash
awslocal s3 mb s3://algashop-product-image || true
awslocal s3api put-bucket-cors --bucket algashop-product-image --cors-configuration file:///etc/aws/cors.json
awslocal s3 sync /etc/images s3://algashop-product-image
```

Três detalhes carregam o peso:

- **`awslocal`, não `aws`** — o wrapper já aponta para o endpoint do próprio container. O `aws` puro tentaria falar com a AWS de verdade.
- **`|| true`** — o script roda de novo a cada recriação do container, e criar bucket que já existe é erro. Sem isso, o erro aborta o resto do arquivo e o CORS nunca é aplicado.
- **`s3 sync` num processo só** — a versão anterior subia os 23 arquivos em paralelo e estourava a memória do container; o OOM killer derrubava o LocalStack no meio da inicialização. É por isso que o limite aqui é **1 GB**, o dobro dos outros serviços de apoio.

### CORS não é detalhe

```json
{ "CORSRules": [{ "AllowedHeaders": ["*"], "AllowedMethods": ["GET","POST","PUT"],
                  "AllowedOrigins": ["*"], "ExposeHeaders": ["ETag"] }] }
```

É o CORS que autoriza o **navegador** a fazer o `PUT` direto. Sem essa regra o upload falha no *preflight*, antes de sair um byte — e a mensagem no console não menciona S3 nenhum, o que torna o sintoma difícil de ligar à causa.

`AllowedOrigins: ["*"]` é aceitável em estudo e não seria em produção: qualquer página da internet poderia postar no bucket, desde que tivesse uma URL assinada válida.

### A pegadinha do hostname

A URL assinada carrega o endpoint configurado e vai **para o navegador**:

```
http://algashop-localstack:4566/algashop-product-image/4425a4f1-....jpg?X-Amz-Algorithm=...
```

`algashop-localstack` é nome de container — a máquina do desenvolvedor não resolve. Daí as três linhas novas em `etc/hostnames/hostnames`:

```
127.0.0.1 algashop-localstack
127.0.0.1 s3.algashop-localstack
127.0.0.1 algashop-product-image.algashop-localstack
```

As duas últimas cobrem o estilo *virtual-hosted* (`bucket.host`), caso `path-style-access-enabled` seja desligado.

> É a consequência direta de "não passa pelo backend": **o cliente precisa resolver o mesmo nome que o servidor usou para assinar**. Em produção isso desaparece — o endpoint vira o da AWS, que a internet já resolve. É o tipo de coisa que só quebra na hora da demonstração.

---

## Health check

```java
@Component("awsS3")
@ConditionalOnProperty(name = "algashop.storage.provider", havingValue = "s3", matchIfMissing = true)
public class StorageProviderAwsS3HealthIndicator implements HealthIndicator {
    public Health health() {
        return storageProviderAwsS3.healthCheck() ? Health.up().build()
                                                  : Health.status("DEGRADED").build();
    }
}
```

Terceiro indicador do projeto, e **ele funciona** — ao contrário do de cache, documentado em [health check](../04-infraestrutura/health-checks.md). Verificado parando o LocalStack:

| | LocalStack de pé | LocalStack parado |
|---|---|---|
| `awsS3` | `UP` | **`DEGRADED`** |
| agregado | `UP` | `DEGRADED` |
| `mongo` | `UP` | `UP` |
| `/actuator/health/readiness` | `UP` | **`UP`** |
| tempo do `/actuator/health` | 0,011s | 0,30s |

O `readiness` não se mexe, que é o contrato: storage fora não tira a instância de rotação. E o custo de degradar é baixo — 0,3s, porque a conexão é recusada de imediato em vez de esperar timeout. Recuperação em ~5s depois que o LocalStack volta.

> Vale notar o que ele mede: `bucketExists`. É um teste real, com ida à rede — a diferença exata que faltava ao indicador de cache, que reportava `UP` sem abrir conexão.

---

## O preço pago na listagem

Trazer o `mainImage` para o resumo custou o `$project` do pipeline de agregação:

```java
// antes
mongoOperations.aggregate(aggregation, Product.class, ProductSummaryOutput.class)

// depois
List<Product> products = mongoOperations.aggregate(aggregation, Product.class, Product.class)
        .getMappedResults();
products.stream().map(p -> mapper.convert(p, ProductSummaryOutput.class)).toList();
```

O `$project` montava o formato do DTO **no servidor** e calculava `hasDiscount`, `inStock` e `shortDescription` com operadores do Mongo. Ele não sabia produzir um objeto aninhado com URL derivada de uma propriedade da aplicação — então a listagem voltou a materializar o `Product` inteiro e mapear em Java.

Duas consequências, as duas reais:

**O documento completo volta do Mongo** em toda listagem — `description` inteira e o `Set<Image>` inteiro, para exibir um resumo e uma imagem. Numa página de 20 itens, é o custo multiplicado por 20.

**O `shortDescription` mudou de formato em silêncio.** O `$substrCP(0, 50)` cortava em 50 caracteres crus; o `TypeMap` que voltou usa `abbreviate(..., 15)`. Medido na API:

```json
"shortDescription": "15-inch lapt..."
```

12 caracteres e reticências, onde antes vinham 50. Nenhum contrato e nenhum teste afirmam nada sobre esse campo, então nada acusou. Ver [aggregation pipeline](./agregacoes-mongo.md).

Confirmado que o resto sobreviveu à troca: `hasDiscount`, `inStock` e `slug` continuam preenchidos.

---

## Armadilhas

- **Extensão não é conteúdo.** A validação de tipo é por sufixo do nome; o serviço nunca vê os bytes.
- **A URL assinada não limita tamanho.** Provado: 391 KB entregues sob autorização de 100 bytes.
- **O nome do host da URL vai para o cliente.** Se ele não resolver esse nome, o upload falha no navegador com o serviço perfeitamente saudável.
- **CORS ausente falha no preflight**, com mensagem que não menciona S3.
- **`fileExists` antes de assinar nunca dá positivo** — a chave é um UUID recém-sorteado. É uma ida ao S3 em todo pedido de upload, por uma condição impossível.
- **`@CacheEvict` cobre `PRODUCTS`, não a listagem.** Hoje não há bug porque a listagem não é cacheada; mas o `mainImage` agora aparece no resumo, então cachear a listagem passa a exigir evict aqui.
- **O perfil `production` não sobe.** O grupo é `base + production-env`, e `bucket-name`, `image-storage-url` e o endpoint moram em `development-env`.

---

## Pendências registradas

- [ ] **Nenhum teste cobre imagens ou storage.** Zero. Nem o `Product.addImage`/`removeImage` (que é domínio puro e testável sem infraestrutura nenhuma), nem os application services com o `StorageProviderFakeImpl` — que existe exatamente para isso e não é usado por teste algum.
- [ ] **Objetos órfãos.** Entre pedir a URL e reivindicar a imagem, o arquivo pode ficar no bucket sem dono, e nada o recolhe. Um *lifecycle rule* no bucket ou uma varredura periódica resolveria.
- [ ] **Tamanho não é imposto.** Migrar para POST com policy e `content-length-range`, ou aceitar e limitar por cota do bucket.
- [ ] **Sem autenticação no `/api/v1/upload-requests`** — endpoint aberto que emite permissão de escrita. É o mesmo buraco de todo o projeto, com consequência nova.
- [ ] **Conteúdo nunca é inspecionado.** Nada garante que o objeto seja uma imagem. Exigiria etapa assíncrona pós-upload.
- [ ] **A listagem voltou a trafegar o documento inteiro**, e o `shortDescription` mudou de 50 para 15 caracteres sem que nenhum contrato notasse.
- [ ] **`delete` remove do storage antes de salvar o agregado.** Se o `save` falhar depois do `deleteFile`, o arquivo já foi e o produto continua apontando para ele.
- [ ] **`FileReference` não valida `contentLength` nulo** — o `<= 0` desempacota e estoura `NullPointerException`. Hoje não acontece porque o `@NotNull` do input barra antes.

---

## Checklist de revisão

- [ ] O arquivo passa pelo backend em algum ponto?
- [ ] O nome do arquivo no storage é gerado pelo servidor, e não pelo cliente?
- [ ] A URL assinada tem prazo curto?
- [ ] Existe conferência de que o arquivo chegou, antes de referenciá-lo no domínio?
- [ ] O host da URL assinada é resolvível **pelo cliente**?
- [ ] O CORS do bucket permite o método que o navegador vai usar?
- [ ] Alguma coisa recolhe arquivos órfãos?
- [ ] A URL de leitura é derivada, e não persistida?

---

## Referências

- [Spring Cloud AWS — S3](https://docs.awspring.io/spring-cloud-aws/docs/current/reference/html/index.html#s3-integration)
- [AWS — Presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html)
- [LocalStack — Init hooks](https://docs.localstack.cloud/references/init-hooks/)
- [Ports & Adapters](../01-arquitetura-design/ports-hexagonal.md) — a porta e os dois adapters
- [Threads e concorrência](../04-infraestrutura/threads-e-concorrencia.md) — por que upload pelo backend não escala
- [Health check e degradação](../04-infraestrutura/health-checks.md) — o indicador `awsS3`
