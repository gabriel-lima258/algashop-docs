# Identidade, credenciais e os fundamentos do OAuth 2

> Antes de configurar qualquer coisa: que tipos de credencial existem e o que cada uma resolve, a diferença entre autenticar e autorizar, os quatro papéis do OAuth 2, e por que escopo não é o mesmo que perfil de usuário.
> Este documento é conceitual. A configuração real do projeto está em [Authorization Server](./authorization-server.md).

> Dezenove fases construíram um sistema que **não sabe quem está do outro lado**. "Autenticação" aparece como pendência em quase todos os documentos desde a Fase 8 — a auditoria do catálogo grava um `UUID` aleatório como autor, e o endpoint que emite URLs de escrita no S3 é aberto. Esta trilha começa a fechar isso.

---

## Três coisas que provam identidade

Elas costumam ser tratadas como intercambiáveis, e não são. Cada uma responde a uma pergunta diferente sobre **quem valida, quanto dura e o que acontece quando vaza**.

| | Senha | Certificado | Token |
|---|---|---|---|
| O que é | segredo que alguém memoriza | par de chaves, com a pública assinada por uma CA | autorização emitida por um terceiro confiável |
| Quem consegue validar | só quem guarda o hash | **qualquer um** com a cadeia da CA | quem confia no emissor |
| Duração típica | até alguém trocar | meses ou anos | minutos |
| Se vazar | acesso total até a troca | acesso total até a revogação | acesso limitado até expirar |
| Revogar é | fácil (trocar) | difícil (CRL, OCSP) | fácil ou impossível — [depende do formato](./authorization-server.md#opaco--jwt-a-escolha-que-decide-o-resto) |

### O que muda de verdade entre elas

**A senha é um segredo compartilhado.** Para validar, alguém precisa ter algo derivado dela. Isso significa que cada sistema que aceita sua senha é mais um lugar de onde ela pode vazar — e é exatamente por isso que reusar senha entre serviços é perigoso: a segurança do conjunto vira a do elo mais fraco.

**O certificado inverte isso.** A chave privada nunca sai de quem a possui; o que circula é a pública. Qualquer um valida sem nunca ter tido acesso ao segredo. É a propriedade que faz o HTTPS funcionar entre partes que nunca se falaram — e é a mesma que faz o JWT funcionar, como se vê adiante.

**O token resolve um terceiro problema:** carregar adiante uma identificação já feita. Você prova quem é **uma vez**, para quem sabe validar isso, e recebe um comprovante com prazo. Daí a regra que organiza o resto:

> Mandar a senha em toda requisição significa expô-la em toda requisição. O token existe para que a credencial forte seja apresentada **uma vez** e o que circula depois seja algo descartável, limitado e com validade.

E é por isso que **token expira e senha não**. A expiração não é inconveniência: é o mecanismo que limita o estrago de um vazamento, no lugar da revogação que nem sempre existe.

---

## Autenticar não é autorizar

Duas perguntas distintas, que a mesma infraestrutura costuma responder — e confundi-las produz sistemas que sabem quem você é mas não o que você pode, ou o contrário.

| | Pergunta | Exemplo de resposta |
|---|---|---|
| **Autenticação** (*AuthN*) | quem é você? | "é o usuário `gabriel`" |
| **Autorização** (*AuthZ*) | o que você pode fazer? | "pode ler produto, não pode apagar" |

### IdP e IAM

- **IdP** (*Identity Provider*) — o serviço que **autentica**. Ele conhece usuários e credenciais, e é onde a senha realmente é conferida. Google, Keycloak, Entra ID, ou o servidor deste projeto.
- **IAM** (*Identity and Access Management*) — a disciplina, e as ferramentas, de **gerir** quem existe e o que cada um pode. Inclui o IdP, mas também papéis, políticas, grupos e o ciclo de vida de uma conta (criar, promover, desligar).

Na prática de sistemas pequenos os dois moram no mesmo servidor, e por isso são fáceis de confundir. A separação importa quando cresce: o IdP responde "esta pessoa é quem diz ser"; o IAM responde "esta pessoa, hoje, neste contexto, pode fazer isto".

> **Terceirizar a autenticação é quase sempre certo. Terceirizar a autorização quase nunca é.** Quem é o usuário é um fato global; o que ele pode fazer depende de regras do seu domínio, e essas regras não cabem num provedor genérico.

---

## O problema que o OAuth resolve

O OAuth não nasceu para "fazer login". Nasceu para resolver **delegação sem compartilhar segredo**.

O exemplo canônico: você quer que um site de impressão acesse suas fotos guardadas em outro serviço. A solução ingênua é dar a sua senha ao site de impressão. Isso é ruim em quatro dimensões de uma vez:

1. Ele passa a poder fazer **tudo** que você pode, não só ver fotos
2. Ele passa a poder fazer isso **para sempre**, não só agora
3. Para revogar, você precisa **trocar a senha** — e quebrar todos os outros lugares
4. Você não tem como saber **o que ele fez**

O OAuth troca isso por: você se autentica **no serviço de fotos**, autoriza ali um acesso específico, e o site de impressão recebe um **token** que só serve para ler fotos, só por um tempo, e que pode ser cancelado sozinho.

> A senha nunca sai do lugar onde ela vale. O que circula é uma autorização limitada, emitida por quem tinha o direito de emiti-la.

Esse desenho é o mesmo quando não há usuário nenhum e são dois serviços conversando — que é o caso deste projeto hoje.

---

## Os quatro papéis

| Papel | Quem é | Neste projeto |
|---|---|---|
| **Resource Owner** | quem é dono do dado e autoriza o acesso | *ainda ninguém* — não há usuário no fluxo atual |
| **Client** | quem **pede** o token | `algashop-ordering` querendo ler o catálogo |
| **Authorization Server** | quem **emite** o token | `algashop-authorization-server` |
| **Resource Server** | quem **exige e valida** o token | `product-catalog`, `ordering`, `billing` — *nenhum configurado ainda* |

Duas confusões valem ser desfeitas de saída:

> ⚠️ **"Client" não quer dizer "aplicativo do usuário final".** Client é **quem pede o token** — pode ser um app de celular, um SPA, ou um microsserviço no backend. Neste projeto o `ordering` é um client: ele pede token para poder chamar o catálogo.

> ⚠️ **Authorization Server e Resource Server são papéis, não necessariamente processos separados.** Eles podem viver no mesmo serviço. Separá-los é o que faz sentido aqui, e a razão é a próxima seção.

### Emitir e verificar são responsabilidades diferentes

É a decisão que organiza todo o resto:

```
                 credenciais            token
   client  ──────────────────▶  AUTHORIZATION SERVER
      │                                  │
      │  ◀─────── token ─────────────────┘
      │
      │  Authorization: Bearer <token>
      └──────────────────────────▶  RESOURCE SERVER
                                         │
                              valida (assinatura ou introspecção)
```

Quem verifica **nunca vê a senha**. Isso tem três consequências práticas:

- **O segredo mora num lugar só.** Auditar, rotacionar e proteger a credencial forte vira problema de um serviço, não de cinco.
- **Um resource server comprometido não entrega credenciais.** Ele só teve tokens de curta duração passando por ele.
- **Adicionar um serviço não aumenta a superfície de autenticação.** O novo serviço aprende a validar token; ele não aprende a validar senha.

---

## Grant types: como se chega ao token

*Grant type* é o **procedimento** para obter um token. Cada um existe para uma situação diferente, e escolher errado é uma falha de segurança, não de estilo.

| Grant | Quando usar | Há usuário? |
|---|---|---|
| **`client_credentials`** | serviço falando com serviço | **não** |
| **`authorization_code`** (+ PKCE) | app ou site agindo em nome de uma pessoa | sim |
| **`refresh_token`** | renovar sem pedir login de novo | sim |
| `device_code` | TV, console, coisas sem teclado | sim |
| ~~`implicit`~~ | — | **removido no OAuth 2.1** |
| ~~`password`~~ | — | **removido no OAuth 2.1** |

### Por que este projeto começou por `client_credentials`

Porque é o único caso que existe hoje: **não há usuário no fluxo**. O `ordering` precisa ler o catálogo por conta própria, não em nome de ninguém. O client se autentica com o próprio segredo e recebe um token que representa *ele mesmo*.

É também o grant mais simples — uma requisição, sem redirecionamento, sem navegador, sem tela de consentimento — e por isso o certo para começar. Todo o vocabulário (client, escopo, token, expiração, formato) aparece nele sem o ruído do fluxo com usuário.

### `authorization_code` + PKCE, o próximo passo

É o fluxo com pessoa: o usuário é redirecionado ao authorization server, autentica-se **lá**, consente, e o client recebe de volta um **código** que troca por token. O código existe para que o token nunca trafegue pela barra de endereço.

O **PKCE** fecha o buraco que sobra: o client gera um segredo aleatório (`code_verifier`), manda o hash dele (`code_challenge`) na ida, e o segredo cru na troca. Quem interceptar o código não consegue usá-lo sem o verificador.

> **Por que o `implicit` morreu:** ele devolvia o token direto na URL de redirecionamento, e URL vaza — fica no histórico, no `Referer`, no log do proxy. Ele existia porque navegadores antigos não deixavam um SPA fazer a troca; com CORS e PKCE isso deixou de ser verdade, e o OAuth 2.1 o removeu.
>
> **Por que o `password` morreu:** ele pedia que o client coletasse a senha do usuário e a repassasse ao authorization server — exatamente o que o OAuth existe para evitar.

---

## Escopo é teto, não papel

Escopo (*scope*) é o **limite máximo** do que aquele token pode fazer. `products:read` não diz quem o portador é; diz que, com este token, só dá para ler produto.

```yaml
scopes:
  - products:read
  - products:write
```

A distinção que mais confunde:

| | Responde | Quem define | Vive |
|---|---|---|---|
| **Escopo** | o que **este token** pode | quem pediu o token, limitado pelo cliente | no token |
| **Role / papel** | o que **esta pessoa** é | o seu domínio | no seu banco |
| **Permissão** | o que **é permitido agora** | a sua regra de negócio | na sua decisão |

> ⚠️ **Escopo não substitui autorização de domínio.** Um token com `products:write` diz que o portador *pode ter permissão de escrever* — não que ele possa escrever **naquele** produto. "Só o dono edita o próprio anúncio" é regra sua, e nenhum escopo a expressa.

O caminho correto é os dois em série: o escopo **estreita** o que é possível, e a regra de negócio decide o resto. Um token restrito nunca ganha poder por causa de uma regra permissiva; um token amplo ainda pode ser barrado pela regra. A ordem importa, e o escopo vem primeiro porque é mais barato — dá para negar antes de tocar no banco.

E note o desenho dos dois clientes deste projeto: o de teste tem leitura **e** escrita; o do `ordering` tem só leitura. **O escopo mais estreito que faz o trabalho é o certo** — se o `ordering` só lê, um token dele que vaze não escreve.

---

## O que mudou no OAuth 2.1

O 2.1 não é um protocolo novo: é a consolidação de quinze anos de erratas e boas práticas do 2.0, com o que se provou perigoso **removido**.

| Mudança | Por quê |
|---|---|
| **PKCE obrigatório** em `authorization_code` | interceptação de código deixou de ser hipótese |
| **`implicit` removido** | token na URL vaza por design |
| **`password` removido** | o client não deve ver a senha do usuário |
| **Redirect URI por comparação exata** | correspondência por prefixo permitia redirecionamento aberto |
| **Refresh token rotacionado ou vinculado** | limita o estrago de um refresh vazado, que é o token mais valioso |

> A lição transferível é a direção: **as mudanças quase todas retiram opções.** Um protocolo com muitas alternativas é um protocolo em que alguém vai escolher a errada, e a segurança do conjunto passa a depender de quem lê a documentação até o fim.

---

## Checklist de revisão

- [ ] A credencial forte (senha, segredo do cliente) é apresentada **uma vez**, e não em toda requisição?
- [ ] Quem valida o token precisa conhecer a senha? *(deveria ser não)*
- [ ] O token tem prazo curto o bastante para que expirar substitua revogar?
- [ ] O escopo pedido é o **mínimo** que faz o trabalho?
- [ ] A regra de negócio decide depois do escopo, e não em vez dele?
- [ ] O grant escolhido corresponde a haver ou não usuário no fluxo?
- [ ] Se há usuário, o fluxo é `authorization_code` **com PKCE**?

---

## Referências

- [RFC 6749 — The OAuth 2.0 Authorization Framework](https://datatracker.ietf.org/doc/html/rfc6749)
- [OAuth 2.1 (draft)](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1)
- [RFC 7636 — PKCE](https://datatracker.ietf.org/doc/html/rfc7636)
- [OAuth 2.0 Security Best Current Practice](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics)
- [Authorization Server](./authorization-server.md) — a configuração real deste projeto
