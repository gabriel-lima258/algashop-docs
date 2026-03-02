# algashop-docs

## CQS e CQRS

### CQS — Command Query Separation

CQS é um **princípio de design** criado por Bertrand Meyer que estabelece que todo método de um objeto deve ser classificado em uma de duas categorias:

- **Command (Comando)**: executa uma ação que altera o estado do sistema. **Não retorna valor** (retorna `void`).
- **Query (Consulta)**: retorna dados ao chamador. **Não altera o estado** do sistema (é livre de efeitos colaterais).

Em resumo: **"Fazer uma pergunta não deve mudar a resposta."**

#### Exemplo em código

```java
// Command — altera estado, não retorna nada
void debitAccount(BigDecimal amount);

// Query — retorna dado, não altera estado
BigDecimal getBalance();

// Violação de CQS — altera estado E retorna valor
BigDecimal debitAndReturnBalance(BigDecimal amount);
```

---

### CQRS — Command Query Responsibility Segregation

CQRS é um **padrão arquitetural** que aplica a ideia do CQS em nível de **arquitetura da aplicação**, separando os modelos de leitura e escrita em responsabilidades distintas.

- **Command Model (Write Model)**: responsável por receber comandos, validar regras de negócio e persistir mudanças de estado. Usa o modelo de domínio rico (entidades, agregados, value objects).
- **Query Model (Read Model)**: responsável por consultas otimizadas para leitura. Pode usar projeções desnormalizadas, views materializadas ou até bancos de dados diferentes.

#### Abordagens de CQRS

| Abordagem | Descrição |
|-----------|-----------|
| **CQRS simples (mesmo banco)** | Separa os modelos de comando e consulta em nível de código, mas ambos usam o mesmo banco de dados. É a forma mais comum e mais fácil de adotar. |
| **CQRS com bancos separados** | O Write Model persiste em um banco (ex: PostgreSQL) e o Read Model consulta outro banco otimizado para leitura (ex: Elasticsearch, Redis). Requer sincronização entre os bancos. |
| **CQRS com Event Sourcing** | Em vez de persistir o estado atual, persiste todos os eventos de domínio. O Read Model é reconstruído a partir da projeção desses eventos. É a abordagem mais complexa, mas oferece auditoria completa e replay de eventos. |

---

### O que NÃO confundir

| | CQS | CQRS |
|---|-----|------|
| **O que é** | Princípio de design de métodos | Padrão arquitetural |
| **Escopo** | Nível de método/objeto | Nível de aplicação/sistema |
| **Separação** | Métodos que leem vs. métodos que escrevem | Modelos inteiros de leitura vs. escrita |
| **Complexidade** | Simples, aplicável em qualquer código | Adiciona complexidade arquitetural |
| **Quando usar** | Sempre que possível como boa prática | Quando há necessidades distintas de leitura e escrita (ex: alta escala de leitura, modelos de consulta complexos) |

- **CQS não exige CQRS**: você pode seguir CQS em métodos sem separar modelos.
- **CQRS se inspira no CQS**: mas vai além, separando responsabilidades em nível arquitetural.
- **CQRS não exige Event Sourcing**: é perfeitamente válido usar CQRS com bancos relacionais tradicionais.
- **Event Sourcing não exige CQRS**: mas os dois são frequentemente combinados porque Event Sourcing naturalmente gera um Write Model baseado em eventos que precisa de projeções para leitura eficiente.