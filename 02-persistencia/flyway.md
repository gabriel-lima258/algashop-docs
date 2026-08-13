# Flyway - Migrações de Banco de Dados

## O que é o Flyway?

O Flyway é uma ferramenta de versionamento e migração de banco de dados. Ele permite que você versione o schema do banco da mesma forma que versiona o código-fonte, garantindo que todos os ambientes (dev, staging, produção) tenham a mesma estrutura de banco.

O Flyway funciona através de **scripts SQL versionados** (ex: `V1__create_table_customer.sql`, `V2__add_column_email.sql`) que são aplicados em ordem. Ele mantém uma tabela interna (`flyway_schema_history`) para saber quais migrações já foram executadas.

---

## Configuração do Plugin no Gradle - ótimo em produção

### `build.gradle`

```groovy
// O bloco buildscript define dependências que o próprio Gradle precisa
// para executar plugins. O Flyway precisa do driver do PostgreSQL
// e do módulo flyway-core disponíveis no classpath do build (não da aplicação).
buildscript {
    dependencies {
        // Biblioteca core do Flyway - contém o motor de migrações
        classpath 'org.flywaydb:flyway-core:11.13.0'

        // Driver específico para PostgreSQL.
        // A partir do Flyway 10+, o suporte a cada banco foi separado
        // em módulos independentes. Sem isso, o Flyway não consegue
        // se conectar ao PostgreSQL e você recebe:
        // "No database found to handle jdbc:postgresql://..."
        classpath 'org.flywaydb:flyway-database-postgresql:11.13.0'
    }
}

plugins {
    // Registra o plugin do Flyway, que adiciona as tasks:
    // flywayMigrate, flywayInfo, flywayValidate, flywayClean, flywayRepair, etc.
    id 'org.flywaydb.flyway' version '11.13.0'
}
```

### Por que `buildscript` E `plugins`?

- **`plugins {}`** — registra o plugin e disponibiliza as tasks (`flywayMigrate`, `flywayInfo`, etc.)
- **`buildscript { dependencies {} }`** — coloca as bibliotecas no classpath do Gradle para que o plugin consiga executar. O bloco `plugins` sozinho não carrega o driver do PostgreSQL, por isso o `buildscript` é necessário.

Se você usar apenas o `plugins {}` sem o `buildscript`, vai receber o erro:
```
No database found to handle jdbc:postgresql://...
```

---

## Arquivo de Configuração (`flyway.conf`)

O arquivo `flyway.conf` fica na raiz do projeto e contém as credenciais e configurações de conexão:

```properties
# URL de conexão com o banco
flyway.url=jdbc:postgresql://localhost:5433/ordering

# Credenciais
flyway.user=postgres
flyway.password=postgres

# Diretório onde ficam os scripts de migração (relativo ao projeto)
# Por padrão, o Flyway procura em src/main/resources/db/migration
flyway.locations=filesystem:src/main/resources/db/migration
```

O parâmetro `-Dflyway.configFiles=flyway.conf` nos comandos diz ao Flyway para ler esse arquivo em vez de usar apenas as configurações do `build.gradle`.

---

## Comandos

### `flywayInfo` — Consultar estado das migrações

```bash
./gradlew flywayInfo -Dflyway.configFiles=flyway.conf
```

**O que faz:** Lista todas as migrações encontradas e seus estados (Pending, Applied, Failed). Não altera nada no banco — é apenas uma consulta.

**Quando usar:** Para verificar quais migrações já foram aplicadas e quais estão pendentes antes de rodar o `flywayMigrate`.

**Exemplo de saída:**
```
+-----------+---------+---------------------+----------+---------------------+----------+----------+
| Category  | Version | Description         | Type     | Installed On        | State    | Undoable |
+-----------+---------+---------------------+----------+---------------------+----------+----------+
| Versioned | 1       | create table order  | SQL      | 2026-03-20 15:30:00 | Success  | No       |
| Versioned | 2       | add column status   | SQL      |                     | Pending  | No       |
+-----------+---------+---------------------+----------+---------------------+----------+----------+
```

---

### `flywayValidate` — Validar integridade das migrações

```bash
./gradlew -x check build flywayValidate -Dflyway.configFiles=flyway.conf
```

**O que faz:** Compara os scripts de migração locais com o que foi registrado no banco (`flyway_schema_history`). Verifica se algum script já aplicado foi alterado (checksum diferente) ou removido.

**Quando usar:** Antes de fazer deploy, para garantir que ninguém alterou um script de migração que já foi aplicado. Alterações em migrações já executadas quebram a integridade do versionamento.

**Erros comuns detectados:**
- Script já aplicado foi editado (checksum diferente)
- Script já aplicado foi deletado
- Gap na sequência de versões

---

### `flywayMigrate` — Executar migrações pendentes

```bash
./gradlew -x check build flywayMigrate -Dflyway.configFiles=flyway.conf
```

**O que faz:** Executa todos os scripts de migração pendentes em ordem de versão. Cada script é executado uma única vez e registrado na tabela `flyway_schema_history`.

**Quando usar:** Para aplicar novas migrações ao banco (criar tabelas, adicionar colunas, inserir dados, etc.).

**Importante:** Este comando **altera o banco de dados**. Sempre rode `flywayInfo` antes para conferir quais migrações serão aplicadas.

---

## O flag `-x check build`

Nos comandos `flywayValidate` e `flywayMigrate`, usamos:

```bash
./gradlew -x check build flywayMigrate ...
```

- **`build`** — compila o projeto antes de executar a task do Flyway. Isso garante que os arquivos de migração em `src/main/resources` sejam processados e copiados para o classpath.
- **`-x check`** — exclui a task `check` (que roda os testes). Sem isso, o Gradle executaria todos os testes antes de rodar o Flyway, o que é desnecessário quando você só quer migrar o banco.

No `flywayInfo` não precisamos do `build` porque ele apenas lê os arquivos de migração diretamente, sem depender do classpath compilado.

---

## Estrutura dos Scripts de Migração

Os scripts ficam em `src/main/resources/db/migration/` e seguem a convenção de nomenclatura:

```
V{versão}__{descricao}.sql
```

Exemplos:
```
V1__create_table_customer.sql
V2__create_table_order.sql
V3__add_column_email_to_customer.sql
```

**Regras:**
- O prefixo `V` indica migração versionada
- O número da versão deve ser sequencial
- **Dois underscores** (`__`) separam a versão da descrição
- Uma vez aplicado, o script **nunca deve ser editado**. Crie um novo script para alterações

---

## Um caso diferente: schema ditado por uma biblioteca

O `authorization-server` ganhou Flyway na Fase 23, e o que ele versiona **não é a nossa modelagem**:

```sql
CREATE TABLE oauth2_authorization (
    id varchar(100) NOT NULL,
    registered_client_id varchar(100) NOT NULL,
    ...
```

As duas migrations são cópias fiéis da distribuição do Spring Authorization Server. Nomes de coluna, tipos e tamanhos são lidos pelo `RowMapper` da biblioteca: renomear uma coluna ou apertar um `varchar` quebra a persistência **em runtime**, não em compilação.

> Nos outros serviços a migration registra uma decisão nossa, e mudá-la é uma decisão nova. Aqui ela registra um **contrato de terceiro** — e o versionamento serve para acompanhar a biblioteca, não para evoluir o modelo.

E um lembrete que custou um erro para aprender: **não se edita migration já aplicada, nem para acrescentar comentário.** Prefixar um cabeçalho explicativo no `.sql` muda o checksum, e o Flyway recusa subir:

```
FlywayValidateException: Validate failed: Migrations have failed validation
```

A explicação pertence à documentação; o arquivo aplicado é imutável. Ver [Authorization code e consentimento](../05-seguranca/authorization-code-e-consentimento.md).

---

## Resumo Rápido

| Comando | Altera o banco? | Quando usar |
|---------|:---:|---|
| `flywayInfo` | Não | Ver estado atual das migrações |
| `flywayValidate` | Não | Verificar integridade antes de deploy |
| `flywayMigrate` | Sim | Aplicar migrações pendentes |
