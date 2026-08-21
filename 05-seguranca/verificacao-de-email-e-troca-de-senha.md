# Verificação de e-mail e troca de senha

A Fase 25 deixou uma pendência constrangedora: o cadastro de usuário gerava uma senha aleatória, **imprimia no stdout** e guardava só o hash. Não havia e-mail, não havia retorno no corpo — o usuário criado pela API simplesmente **nunca conseguia logar**.

Esta fase fecha isso, e o caminho revela algo mais interessante que "mandar e-mail": **definir a primeira senha e recuperar a senha esquecida são a mesma operação**, e por isso são o mesmo código.

---

## A ideia central: um token que o banco não sabe ler

```
1. o servidor sorteia  plainToken = "oyGGaynCrWeoLMHJEVRXSNBk"   (24 caracteres)
2. guarda no banco     SHA256(plainToken) em Base64 URL-safe
3. manda no e-mail     .../change-password?token=oyGGaynCrWeoLMHJEVRXSNBk
4. quando o link volta  recalcula o hash e compara
```

Verificado, lado a lado:

```
token no e-mail:  oyGGaynCrWeoLMHJEVRXSNBk
token no banco:   v4-7MyCVb9v4GVWguYmryRHYX95qjTbVK8vf5nw2j54
```

> **O texto puro existe uma vez só, no instante em que sai para o e-mail.** Quem lê a tabela — um DBA, um dump vazado, um backup — não consegue reconstruir nenhum link.

É a mesma forma do PKCE (Fase 26): o segredo viaja, o resumo fica. A diferença é o sentido — no PKCE quem guarda o hash é o servidor por um instante; aqui é o banco, por horas.

### Por que SHA-256 e não BCrypt

Senha usa BCrypt; token de verificação, não. Três razões:

| | Senha | Token de verificação |
|---|---|---|
| Origem | escolhida por humano, previsível | 24 caracteres aleatórios |
| Vida | anos | 2 a 24 horas |
| Precisa ser **buscável**? | não (busca por e-mail) | **sim** — `findByVerificationToken(hash)` |

A terceira é a que decide. BCrypt gera um **salt novo a cada chamada**: cifrar o mesmo token duas vezes dá resultados diferentes, e nenhuma consulta `WHERE verification_token = ?` funcionaria. Já o custo computacional do BCrypt — sua razão de existir — é desnecessário contra um valor aleatório de 24 caracteres, que não tem dicionário a proteger.

```java
return MessageDigest.isEqual(
        hashed.getBytes(UTF_8),
        hash(plainToken).getBytes(UTF_8));
```

`MessageDigest.isEqual` compara em **tempo constante** — não sai no primeiro byte diferente. Contra ataque de temporização, que mediria quantos caracteres do token acertou.

---

## O agregado orquestra; a aplicação só encaminha

Esta é a parte que dá nome à fase. Compare as duas metades:

```java
// PasswordManagementApplicationService — busca, chama, salva
public void changePasswordWithToken(String plainToken, String newPlainPassword) {
    String hash = tokenHasher.hash(plainToken);
    AuthUser user = authUserRepository.findByVerificationToken(hash)
            .orElseThrow(() -> new AuthUserNotFoundException("..."));
    try {
        user.changePasswordWithToken(plainToken, newPlainPassword, passwordManager, tokenHasher);
    } catch (IllegalArgumentException | IllegalStateException e) {
        throw new AccessDeniedException(e.getMessage());
    }
    authUserRepository.save(user);
}
```

```java
// AuthUser — TODA a regra
public void changePasswordWithToken(String plainToken, String plainPassword,
                                    AuthUserPasswordManager passwordManager,
                                    VerificationTokenHasher tokenHasher) {
    verifyToken(plainToken, tokenHasher);       // confere hash e expiração
    setPassword(passwordManager.encrypt(plainPassword));
    cleanVerificationToken();                   // o link vale UMA vez
    if (!isEmailVerified()) {
        setEmailVerified(true);                 // usar o link É verificar o e-mail
    }
}
```

O serviço de aplicação não decide nada: não sabe quando um token expira, não sabe que usar o link consome o token, não sabe que trocar a senha verifica o e-mail. **Ele traduz uma requisição HTTP em uma chamada de método e uma transação.**

### As duas portas no domínio

O agregado precisa cifrar senha e resumir token — e não pode saber que existe BCrypt ou SHA-256. Daí duas interfaces **dentro** do pacote de domínio:

```java
public interface AuthUserPasswordManager { String generate(); String encrypt(String p); boolean matches(String p, String e); }
public interface VerificationTokenHasher { String generate(); String hash(String t); boolean isEqual(String h, String t); }
```

As implementações ficam na infraestrutura e são **passadas como argumento** para o método do agregado, não injetadas nele. É a variação do padrão que evita dar dependências a uma entidade JPA — o agregado continua sendo um objeto que o Hibernate consegue instanciar, e ainda assim conversa com a política de criptografia.

> Repare no `brandNew`: ele deixou de receber um hash pronto e passou a receber a porta.
> ```java
> user.setPassword(passwordManager.encrypt(passwordManager.generate()));
> ```
> **A senha inicial existe e ninguém a conhece** — nem quem cadastrou, nem quem foi cadastrado. É deliberadamente inútil: só serve para a coluna não nascer nula. Quem define a senha de verdade é o próprio usuário, pelo link.

Foi assim que o `System.out.println(tempPassword)` da Fase 25 morreu — não por ser inseguro, mas por deixar de fazer sentido.

---

## Um fluxo, dois nomes

```
                    ┌─────────────────────────────────────────┐
  POST /api/v1/users│  brandNew  → senha aleatória inútil     │
  (MANAGER)         │  generateVerificationToken(24h)         │
                    │  sendActivationEmail                    │
                    └──────────────────┬──────────────────────┘
                                       │
  POST /forgot-password ───────────────┤        ambos chegam ao mesmo lugar
  (qualquer um)      generateVerificationToken(2h)
                     sendPasswordChangeEmail
                                       │
                                       ▼
                    GET  /change-password?token=…
                    POST /change-password
                         └─ AuthUser.changePasswordWithToken(...)
                              define a senha  +  verifica o e-mail  +  consome o token
```

**"Ativar conta" é "trocar a senha pela primeira vez".** O `if (!isEmailVerified())` é a única linha que distingue os dois — e ela existe só para não regravar um `true` que já é `true`.

Prazos diferentes, mesma mecânica:

| | TTL | Por quê |
|---|---|---|
| ativação | **24h** | a pessoa acabou de ser cadastrada e pode demorar a ver o e-mail |
| troca de senha | **2h** | quem pediu está na frente do computador agora |

---

## A porta que o e-mail fecha: login exige verificação

```java
public boolean isDisabled() {
    return !isEnabled() || !isEmailVerified();
}
```

Duas razões para não entrar, e a segunda é nova. Verificado com um usuário recém-cadastrado:

```
enabled=true  email_verified=false  senha={bcrypt}$2...  token=wqpwGiumvS...
login antes de ativar  ->  /login?error

  ... clica no link, define a senha ...

email_verified=true   token=(nulo)
login depois de ativar ->  http://algashop.local:9080/
```

> **Uma conta pode existir, estar habilitada, ter senha — e ainda assim não servir para entrar.** "Existe" e "pode logar" passaram a ser perguntas diferentes.

O detalhe da migração que isso obriga: `email_verified` nasce `NOT NULL DEFAULT false`, então **todo usuário já existente viraria não verificado** — e nenhum dos quatro do seed conseguiria logar. Por isso o `afterMigrate.sql` marca os quatro como `true` na mesma leva. Migração que muda uma regra de acesso precisa dizer o que fazer com quem já estava lá.

---

## O e-mail, e o que ele não pode fazer

```java
@Async
public void sendPasswordChangeEmail(AuthUser user, String token) { ... }
```

`@Async` para não prender a resposta HTTP no tempo do servidor SMTP, e um `try/catch` em volta do envio para que uma falha de e-mail **não desfaça** a operação que a disparou. As duas decisões apontam para o mesmo princípio: entregar e-mail é efeito colateral, não parte da transação de negócio.

Em desenvolvimento o destino é o **Mailpit** (`docker-compose.tools.yml`, SMTP em 1025, interface em 8025) — um servidor SMTP que aceita tudo e não entrega nada, com caixa de entrada navegável. Verificado:

```
de=no-reply@algashop.com  para=john.doe@email.com  assunto=AlgaShop - Change your password

  Hello John Doe,
  Use the link bellow to change your password:
  http://auth.algashop.local:9000/change-password?token=VRUBLjJqAMMlsPrVMmrfiyKY
  This link expires in 2 hour(s)
```

### O bug do prazo prometido

O e-mail de troca de senha formatava a duração com `getActivationTtl()` — **o TTL do outro fluxo**. O token expirava em 2 horas e a mensagem prometia 24. Ninguém notaria até alguém tentar usar o link no dia seguinte e receber "Invalid token." com o e-mail dizendo que ainda valia.

> **Copiar um método e trocar metade das linhas é o jeito mais comum de produzir uma mentira consistente.** As duas mensagens são quase idênticas; a diferença que importava estava numa chamada de getter.

---

## Não dizer quem existe

Duas telas deste fluxo poderiam responder à pergunta *"esta pessoa tem conta no AlgaShop?"*, e nenhuma responde.

**No `/forgot-password`**, `requestPasswordChange(email)` lança `AuthUserNotFoundException` quando o e-mail não existe. Deixar a exceção subir daria uma resposta diferente para e-mail cadastrado e não cadastrado — um oráculo de enumeração. A resposta é sempre a mesma:

```
john.doe@email.com  -> 200  "If an account exists for that address, we have sent a link…"
ninguem@x.com       -> 200  "If an account exists for that address, we have sent a link…"   (e nenhum e-mail sai)
```

É a mesma decisão que o login já tinha tomado na Fase 24 (usuário inexistente e senha errada respondem igual), agora aplicada ao outro lugar onde ela cabia.

**No `/change-password`**, os três jeitos de o token estar errado precisam responder igual:

| | Antes | Depois |
|---|---|---|
| token inexistente | **404** `application/problem+json` — *"User not found by verification token"* | 200, `Invalid token.` |
| token já usado | **404** idem | 200, `Invalid token.` |
| token expirado | 200, `Invalid token.` | 200, `Invalid token.` |

O controller só capturava `AccessDeniedException`. Como um token errado nem chega a ser encontrado, ele escapava como `AuthUserNotFoundException` e caía no `ApiExceptionHandler` — que devolvia um **JSON de API dentro de uma tela de navegador**, confirmando de quebra que aquele token não existe.

> A parte instrutiva: o `catch` estava lá, com a mensagem certa, e **quase nunca era alcançado**. Ele só pegava o caso de token *encontrado e expirado*. Um tratamento de erro pode existir, parecer completo, e cobrir o caminho menos provável.

---

## O achado que não deu para observar

`requestPassword` faz, nesta ordem, dentro de um método `@Transactional`:

```java
String plainToken = user.generateVerificationToken(ttl, tokenHasher);
emailSender.sendPasswordChangeEmail(user, plainToken);   // @Async — sai AGORA
authUserRepository.save(user);                            // commit só no fim do método
```

O e-mail é despachado para outra thread **antes de o token estar gravado**. Entre o `send` e o commit existe uma janela em que o link já está na caixa de entrada e `findByVerificationToken` ainda não encontra nada — o usuário receberia "Invalid token." num link legítimo.

Não consegui observar isso: a janela é de milissegundos e depende de o usuário clicar mais rápido que o commit. Mas ela é **estrutural**, não probabilística — e o mesmo padrão está no cadastro (`create()` envia antes de salvar).

O conserto idiomático seria publicar um evento de aplicação e ouvir com `@TransactionalEventListener(phase = AFTER_COMMIT)` — que é justamente o mecanismo que o `ordering` já usa para eventos de domínio ([`eventos-e-listeners.md`](../01-arquitetura-design/eventos-e-listeners.md)). Fica registrado como pendência, porque a mudança é de desenho.

> **`@Async` dentro de `@Transactional` publica antes do commit.** É a versão "efeito colateral" do problema clássico de ler dados que ainda não existem.

---

## O que mudou na configuração

```yaml
algashop:
  user-account:
    token:
      activation-ttl: PT24H
      password-reset-ttl: PT2H
    mail:
      from: AlgaShop <no-reply@algashop.com>
      password-change-url: http://auth.algashop.local:9000/change-password
```

E um filter chain novo, antes do padrão:

```java
@Bean @Order(2)
public SecurityFilterChain publicSecurityFilterChain(HttpSecurity http) {
    http.securityMatcher("/change-password", "/forgot-password")
            .authorizeHttpRequests(authorize -> authorize.anyRequest().permitAll())
            .sessionManagement(session -> session.sessionCreationPolicy(STATELESS))
            .requestCache(RequestCacheConfigurer::disable)
            .anonymous(AbstractHttpConfigurer::disable);
    return http.build();
}
```

Três decisões, e nenhuma é decorativa:

- **`STATELESS`** — quem esqueceu a senha não deveria criar sessão. A autorização aqui vem do token na URL, não de cookie.
- **`requestCache().disable()`** — sem isso, visitar `/change-password` sem sessão poderia gravar um `SavedRequest`, e o próximo login redirecionaria para a tela de senha em vez do destino certo. É a mesma mecânica de `SavedRequest` descrita em [telas e formulários](./telas-e-formularios-de-login.md#defaultsuccessurl--savedrequest-o-destino-depende-de-como-você-chegou).
- **`anonymous().disable()`** — não há por que criar um `AnonymousAuthenticationToken` numa rota que é pública por natureza.

---

## Armadilhas

- **BCrypt não serve para token buscável** — salt aleatório impede a consulta por hash.
- **Comparação de hash precisa ser em tempo constante** (`MessageDigest.isEqual`).
- **Um TTL copiado do outro fluxo** faz o e-mail prometer o prazo errado.
- **`catch` que só pega o caso raro** — o token *inválido* seguia outro caminho que não o do `catch`.
- **Migração que muda regra de acesso** precisa decidir o que acontece com as linhas existentes.
- **`@Async` dentro de `@Transactional`** dispara antes do commit.
- **Migration aplicada não se edita**, nem para comentar. *(Aconteceu de novo nesta fase — segunda vez no projeto.)*

## Pendências registradas

- [ ] **O e-mail sai antes do commit** — mover para `@TransactionalEventListener(AFTER_COMMIT)`.
- [ ] **Nenhum teste cobre este fluxo** — token, expiração, reuso e ativação foram provados só por `curl`.
- [ ] **Não há reenvio de link** nem limite de pedidos: `POST /forgot-password` pode ser chamado à vontade, gerando um token novo (e invalidando o anterior) a cada vez. É uma superfície de spam para o e-mail da vítima.
- [ ] **A senha nova não tem regra nenhuma** — nenhum tamanho mínimo, nenhuma validação.
- [ ] **`RandomStringUtils.secure().next(12)`** na senha inicial não restringe o alfabeto; como ela nunca é usada nem exibida, é inofensivo hoje.
- [ ] **Sem confirmação de senha** na tela nem feedback de força.
- [ ] **`POST /api/v1/users/me/password-change`** existe e não tem tela: o usuário logado pode pedir o link, mas só via API.

## Checklist de revisão

- [ ] O token vai em texto puro no e-mail e em hash no banco?
- [ ] A comparação é em tempo constante?
- [ ] O link é de uso único (o token é consumido)?
- [ ] Os três casos de token ruim respondem igual?
- [ ] O e-mail promete o prazo com que o token foi realmente gerado?
- [ ] A migração decidiu o que fazer com os usuários que já existiam?
- [ ] O envio acontece depois do commit?

## Referências

- [OWASP — Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [Mailpit](https://mailpit.axllent.org/)
- [Gestão de usuários e auditoria](./gestao-de-usuarios-e-auditoria.md) · [Telas e formulários de login](./telas-e-formularios-de-login.md) · [PKCE e clientes públicos](./pkce-e-clientes-publicos.md)
