# Scheduled Jobs em Microsserviços

Este documento explica as ferramentas e padrões utilizados para execução de tarefas agendadas (scheduled jobs) em arquiteturas de microsserviços, cobrindo desde o `@Scheduled` do Spring até estratégias distribuídas com controle de concorrência.

---

## O que é um Scheduled Job?

Um **scheduled job** é uma rotina que é executada periodicamente, disparada por tempo (a cada X minutos, todo dia às 03:00, etc.) em vez de ser disparada por uma requisição do usuário.

Exemplos típicos em e-commerce:
- Fechar carrinhos abandonados após 24h
- Gerar faturas no primeiro dia do mês
- Reenviar e-mails de cobrança em atraso
- Sincronizar estoque com ERP externo
- Expirar tokens, limpar logs antigos, gerar relatórios

---

## Ferramentas comuns no ecossistema Spring/Java

### 1. Spring `@Scheduled`

A forma mais simples. Basta anotar um método em um bean Spring:

```java
@Component
@EnableScheduling
public class InvoiceDunningJob {

    @Scheduled(cron = "0 0 3 * * *") // todo dia às 03:00
    public void cobrarFaturasEmAtraso() {
        // ...
    }
}
```

**Prós:** zero configuração, faz parte do Spring Boot.
**Contras:** é **in-process** — se você tem 3 réplicas do microsserviço, o job roda 3 vezes em paralelo. Não há coordenação entre instâncias.

### 2. Quartz Scheduler

Framework mais robusto, com persistência de estado em banco (`QRTZ_*` tables), suporte a jobs duráveis, retries, misfire policies e clustering nativo.

Usar quando:
- Precisa persistir o estado do scheduler (sobreviver a restarts)
- Jobs com parâmetros dinâmicos criados em runtime
- Precisa de cluster mode com coordenação automática entre nós

### 3. ShedLock

**Não é um scheduler** — é uma biblioteca complementar que adiciona **lock distribuído** ao `@Scheduled` do Spring. É a forma mais simples de resolver o problema das N réplicas.

```java
@Scheduled(cron = "0 0 3 * * *")
@SchedulerLock(name = "cobrarFaturasEmAtraso", lockAtMostFor = "10m")
public void cobrarFaturasEmAtraso() { ... }
```

Ele grava uma linha numa tabela `shedlock` (ou Redis/Mongo/ZooKeeper) e só uma instância consegue adquirir o lock por execução.

### 4. Schedulers externos (Kubernetes CronJob, AWS EventBridge, etc.)

Ao invés de embutir o scheduler dentro do microsserviço, você delega ao orquestrador:
- **Kubernetes CronJob**: cria um Pod novo a cada execução
- **AWS EventBridge + Lambda** ou **EventBridge + ECS Task**
- **GCP Cloud Scheduler**

Essa abordagem leva ao padrão de **short-lived microservices**, explicado abaixo.

---

## Short-lived Microservices

Um **short-lived microservice** (microsserviço de vida curta) é um processo que **sobe apenas para executar uma tarefa específica e termina**. Ele não fica de pé 24/7 ouvindo requisições HTTP.

### Como funciona

1. O orquestrador (ex.: Kubernetes CronJob) dispara o container no horário configurado
2. O processo inicia, executa o job (ex.: processar faturas do dia)
3. O processo termina com exit code 0 (sucesso) ou != 0 (falha)
4. O container é destruído

### Por que separar o scheduled do serviço principal?

Misturar jobs agendados dentro do microsserviço que serve requisições HTTP tem vários problemas:

| Problema | Impacto |
|---|---|
| **Acoplamento de recursos** | Um job pesado consome CPU/memória que deveriam atender requisições de usuário |
| **Escalabilidade conflitante** | A API precisa de N réplicas para throughput, mas o job só deveria rodar uma vez |
| **Deploy acoplado** | Uma correção no job força redeploy de toda a API |
| **Observabilidade confusa** | Logs e métricas do job se misturam aos da API |
| **Falhas se propagam** | Um OOM no job derruba quem está servindo usuários |

### Vantagens de extrair para um short-lived service

- **Isolamento total** — o job tem seu próprio container, seus próprios recursos
- **Escala independente** — o scheduler do orquestrador cuida de "quando" rodar
- **Custo menor** — você só paga pelo tempo de execução (importante em serverless)
- **Sem problema de múltiplas réplicas** — o CronJob sobe **um** Pod; não há concorrência acidental
- **Deploy independente** — você versiona o job separadamente da API

### Desvantagens

- Mais artefatos para gerenciar (imagem, pipeline, config)
- Compartilhamento de código com a API exige organização (módulo comum, biblioteca interna)
- Cold start pode ser relevante em jobs muito curtos

---

## Controle de Concorrência com Locks de Banco

Mesmo separando o job em um microsserviço próprio, ainda pode ocorrer execução concorrente:
- O CronJob do Kubernetes pode disparar uma nova execução enquanto a anterior ainda roda (`concurrencyPolicy`)
- Dois ambientes apontando para o mesmo banco (erro humano)
- Retries automáticos

A solução padrão é usar o **próprio banco de dados** como ponto de sincronização, já que ele é a única coisa realmente compartilhada e consistente entre as instâncias.

### 1. Advisory Locks (PostgreSQL)

O PostgreSQL oferece locks aplicacionais que não estão vinculados a linhas:

```sql
-- tenta adquirir sem bloquear; retorna true/false
SELECT pg_try_advisory_lock(123456);

-- libera
SELECT pg_advisory_unlock(123456);
```

O número (`123456`) é um identificador arbitrário do job. Se outra instância tentar o mesmo lock, recebe `false` e desiste. É extremamente barato e não polui tabelas.

### 2. Tabela de Locks (abordagem do ShedLock)

Uma tabela simples para servir como mutex distribuído:

```sql
CREATE TABLE shedlock (
    name        VARCHAR(64)  NOT NULL PRIMARY KEY,
    lock_until  TIMESTAMP    NOT NULL,
    locked_at   TIMESTAMP    NOT NULL,
    locked_by   VARCHAR(255) NOT NULL
);
```

O job tenta um `INSERT` (ou `UPDATE` se `lock_until < now()`). Como a PK é `name`, apenas uma instância consegue. O campo `lock_until` funciona como **lease**: se a instância morrer sem liberar, o lock expira sozinho e outra pode assumir.

### 3. `SELECT ... FOR UPDATE SKIP LOCKED`

Útil quando o job **processa uma fila de itens** no banco (ex.: faturas pendentes) e você quer que múltiplas instâncias trabalhem em paralelo sem pegar o mesmo item:

```sql
SELECT * FROM invoice
 WHERE status = 'PENDING'
 ORDER BY created_at
 LIMIT 100
 FOR UPDATE SKIP LOCKED;
```

O `SKIP LOCKED` faz com que linhas já travadas por outra transação sejam **ignoradas** em vez de causar espera. Isso transforma o banco em uma fila de trabalho segura sem precisar de Kafka/RabbitMQ.

### 4. Optimistic Locking (versionamento)

Quando o risco de concorrência é baixo e você só quer detectar conflitos, usa-se uma coluna `@Version` (JPA). Se duas instâncias tentarem atualizar a mesma linha, a segunda recebe `OptimisticLockException` e pode reprocessar.

---

## Quando escolher qual abordagem

| Cenário | Recomendação |
|---|---|
| Job simples, 1 instância, dev/pequeno | `@Scheduled` puro |
| Várias réplicas da API, job leve embutido | `@Scheduled` + **ShedLock** |
| Job pesado, isolado, roda raramente | **Kubernetes CronJob** (short-lived) |
| Fila de itens processada em paralelo | **`FOR UPDATE SKIP LOCKED`** |
| Scheduler com estado, retries complexos, painel | **Quartz** com JDBC store |
| Serverless, event-driven | **EventBridge + Lambda** / Cloud Scheduler |

---

## Referências

- [Spring Scheduling](https://docs.spring.io/spring-framework/reference/integration/scheduling.html)
- [ShedLock](https://github.com/lukas-krecan/ShedLock)
- [Quartz Scheduler](https://www.quartz-scheduler.org/)
- [PostgreSQL Advisory Locks](https://www.postgresql.org/docs/current/explicit-locking.html#ADVISORY-LOCKS)
- [Kubernetes CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)
