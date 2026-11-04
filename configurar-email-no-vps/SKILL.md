---
name: configurar-email-no-vps
description: >
  Use quando houver e-mail no Postfix/Dovecot de um VPS com CyberPanel
  (Hostinger, AlmaLinux, OpenLiteSpeed): sistema que precisa mandar e-mail
  transacional; caixas para pessoas ou setores com webmail (Roundcube), celular
  e Outlook; domínio que já está no Google Workspace e quer caixas extras no
  servidor; e-mail que "foi enviado" e não chega, cai no spam, log diz 250 e
  ninguém recebe; SPF, DKIM ou DMARC em fail ou softfail; senha certa recusada,
  Outlook que não envia, caixa cheia. Gatilhos: "criar e-mail", "criar caixa",
  "webmail", "e-mail para o setor", "e-mail no servidor e no Google", "split
  delivery", "e-mail não chega", "caiu no spam", "porta 25", "createEmail",
  "nao-responda@", "auditar e-mail". Não use para provedor externo de envio
  (Resend, SendGrid, SES, Brevo) sem Postfix nem caixa no VPS.
---

# E-mail próprio no VPS

Montar o envio de e-mail de um sistema pelo Postfix do próprio servidor, em vez
de contratar provedor. Dá certo, é de graça e a entrega é boa — desde que quatro
coisas estejam certas ao mesmo tempo: a porta de saída, a caixa, as chaves e o
DNS. Errar uma delas normalmente **não** gera erro: gera e-mail que sai, é
aceito, e ninguém lê.

Nesta skill, `<DOMINIO>` é o domínio do remetente e `<HOSTNAME>` é o nome do
servidor (o `hostname -f`, algo como `srv000000.hstgr.cloud`).

## Primeiro: qual destes três trabalhos é

| Pedido | O que ler |
|---|---|
| Um sistema precisa **mandar** e-mail (senha, código, aviso, boleto) | Este arquivo inteiro |
| **Caixas para pessoas** (setor, funcionário) com webmail, celular, Outlook, redirecionamento | Este arquivo até "Os quatro registros", depois `caixas-para-pessoas.md` |
| O domínio **já recebe no Google Workspace** (MX `aspmx.l.google.com`) e quer caixas extras no servidor | `junto-com-google.md` **antes de criar qualquer caixa**: nesse caso o MX não muda e o servidor passa a tratar o domínio como dele, o que quebra e-mail de sistema para quem está no Google |

A medição 7, logo abaixo, diz em qual caso você está. Não pule: quem recebe o
e-mail do domínio hoje decide o desenho inteiro.

## Duas frases que resumem tudo

1. **Meça a porta 25 antes de prometer.** Se ela estiver fechada, nada do resto
   importa, e o sintoma é silêncio.
2. **Aceito não é caixa de entrada.** O `250` prova que o destino recebeu. Não
   prova que a pessoa viu.

O resto deste documento serve a essas duas frases.

## Antes de prometer: sete medições

Rode tudo de uma vez, por SSH, e leia antes de mexer em qualquer coisa:

```bash
echo '--- 1. porta 25 de saida (o furo que mata tudo) ---'
timeout 8 bash -c '</dev/tcp/gmail-smtp-in.l.google.com/25' && echo ABERTA || echo FECHADA

echo '--- 2. os DOIS enderecos do servidor (o mais esquecido) ---'
ip -4 addr show scope global | grep -oP '(?<=inet )[0-9.]+'
ip -6 addr show scope global | grep -oP '(?<=inet6 )[0-9a-f:]+'

echo '--- 3. PTR das duas familias ---'
for ip in $(ip -4 addr show scope global | grep -oP '(?<=inet )[0-9.]+') \
          $(ip -6 addr show scope global | grep -oP '(?<=inet6 )[0-9a-f:]+'); do
  echo -n "$ip -> "; dig +short -x "$ip"
done
hostname -f; postconf -h myhostname

echo '--- 4. listas de bloqueio (troque pelo IPv4 invertido) ---'
# 203.0.113.10 -> 10.113.0.203
for bl in zen.spamhaus.org bl.spamcop.net b.barracudacentral.org dnsbl.sorbs.net; do
  echo -n "$bl: "; dig +short "10.113.0.203.$bl" | grep -q . && echo LISTADO || echo limpo
done

echo '--- 5. quem manda no DNS do dominio ---'
dig +short NS <DOMINIO>

echo '--- 6. os servicos, e se o log ainda esta sendo escrito ---'
systemctl is-active postfix dovecot opendkim | paste -sd' '
ls -la /var/log/maillog; journalctl -u postfix -n 2 --no-pager

echo '--- 7. o que o dominio ja tem hoje (MX, SPF, DMARC, seletores) ---'
dig +short MX <DOMINIO>
dig +short TXT <DOMINIO> | grep -i spf1
dig +short TXT _dmarc.<DOMINIO>
for s in default google selector1 selector2 k1 s1; do
  dig +short TXT "$s._domainkey.<DOMINIO>" | grep -q . && echo "seletor DKIM em uso: $s"
done
postmap -q <DOMINIO> proxy:mysql:/etc/postfix/mysql-virtual_domains.cf && echo 'JA E DOMINIO LOCAL neste Postfix'
```

Como ler cada um:

| Medição | Esperado | Se vier diferente |
|---|---|---|
| Porta 25 | `ABERTA` | **Pare.** Muitos provedores a fecham e nada no painel avisa. Ou abre chamado, ou o caminho é um relay autenticado de terceiro — e aí esta skill não serve |
| Endereços | um IPv4 **e** um IPv6 | Quase todo VPS tem os dois. Os **dois** vão para o SPF. Ver "Os quatro registros" |
| PTR | resolve, e bate com `myhostname` | PTR ausente é rejeição quase certa no Gmail. Quem publica é o dono do bloco de IP, no painel do provedor — não é registro do seu domínio |
| Listas | tudo `limpo` | IP de VPS reaproveitado às vezes vem sujo do inquilino anterior. Pedir remoção antes de começar |
| DNS | o nameserver de verdade | ⚠️ Se a zona não for do PowerDNS local, **mexer no DNS pelo CyberPanel não tem efeito nenhum** e você vai depurar um registro que nunca existiu |
| Serviços | `active` nos três | O OpenDKIM é o que assina. Inativo, o DKIM simplesmente não sai, e o log não reclama |
| Log | `maillog` com data de hoje | Se a data parou, o rsyslog deixou de escrever nele: todo diagnóstico passa a ser `journalctl -u postfix` (e `-u dovecot`). Grep no `maillog` parado mostra um servidor saudável que não existe mais |
| MX de hoje | vazio ou o próprio servidor | MX no Google (`aspmx.l.google.com`) ou em outro provedor: **o MX não muda**, e o caminho é `junto-com-google.md`. Trocar o MX é tirar o e-mail da empresa inteira de onde ele está |
| SPF, DMARC e seletores de hoje | o que existir | Tudo que já existe é **somado**, nunca substituído: um domínio só pode ter **um** SPF, e o DMARC existente (com o `rua=` dele) fica |
| Domínio local | só depois de criar a primeira caixa | Se ele já for local e o MX estiver em outro lugar, os e-mails que os sites deste servidor mandam para esse domínio estão sendo entregues aqui ou recusados. Ver `junto-com-google.md` |

⚠️ **Limite de saída da Hostinger: 5 e-mails por minuto na porta 25.** Passar
disso bloqueia a porta 25 do VPS inteiro, para todos os domínios, até ela reabrir
sozinha (confirmado pelo suporte da Hostinger em 2026). Caixa de pessoa não
serve para disparo em massa, e sistema que manda em volume precisa de freio ou de
relay. Diga isso ao cliente antes de entregar caixas para um setor comercial.

## Os dois saltos, e por que a porta 25 importa mesmo com SMTP autenticado

A pergunta aparece sempre: *"se a aplicação vai usar SMTP autenticado na 587,
por que a 25 importa?"*

São dois saltos, e só o primeiro é autenticado:

1. **Aplicação → Postfix**, porta 587, com usuário e senha. É a entrega do
   texto para quem sabe mandar.
2. **Postfix → servidor do destinatário**, porta 25, sem senha nenhuma. É a
   entrega de verdade, entre servidores.

Como o e-mail mora no VPS, quem faz o segundo salto é ele. Com a 25 fechada, a
mensagem sai da aplicação com sucesso, entra na fila do Postfix e **nunca sai de
lá** — a aplicação recebe "enviado", o log fica quieto, e `mailq` cresce.

Se um dia a entrega passar por um relay externo, a 25 deixa de importar: o relay
faz esse salto.

## O servidor

### A caixa

No CyberPanel a caixa de e-mail depende de o domínio existir como **site**. Se
o domínio do remetente ainda não for um site do painel, ele precisa ser criado
antes — e isso **não mexe em DNS nenhum** (quem é autoritativo é o item 5 das
medições), então um site institucional hospedado em outro lugar continua
intocado.

```bash
# só se o domínio ainda não for um site do painel
cyberpanel createWebsite --package Default --owner admin \
  --domainName <DOMINIO> --email voce@exemplo.com --php 8.3 --ssl 0 --dkim 1 --openBasedir 0

cyberpanel createEmail --domainName <DOMINIO> --userName nao-responda --password '<gerada>'
```

A senha vai direto para o `.env` do servidor, com permissão `600`. Não passa
por lugar nenhum onde fique escrita duas vezes.

### Confira, não confie

Nome de comando e selector mudam entre versões do painel. Dois greps resolvem,
no servidor em que você está trabalhando:

```bash
# quais funções de e-mail este CyberPanel tem
grep -rhoE '"(create|delete|list|change)[A-Za-z]*([Ee]mail|DKIM)[A-Za-z]*"' /usr/local/CyberCP/cli/*.py | sort -u

# qual selector ele usa ao gerar a chave
grep -rhoE 'opendkim-genkey[^"]*' /usr/local/CyberCP/plogical/mailUtilities.py
```

Medido em duas instalações CyberPanel 3.0: as funções são `createEmail`,
`deleteEmail`, `changeEmailPassword`, `listEmailsJson`, e o selector é sempre
**`default`**, fixo no código do painel.

A chave pública fica em `/etc/opendkim/keys/<DOMINIO>/default.txt`, pronta para
o DNS. Confira o mapeamento:

```bash
grep -i <DOMINIO> /etc/opendkim/KeyTable /etc/opendkim/SigningTable
```

A chave que o painel gera tem **1024 bits**. Serve para começar. Para um domínio
que vai ter caixas de pessoas, ou que já assina em outro lugar com seletor
`default`, troque por uma de 2048 bits com seletor próprio (nome curto + ano,
como `emp2026`):

```bash
S=<seletor>   # nome + ano, ex.: emp2026. Nunca "google", que é do Workspace
cd /etc/opendkim/keys/<DOMINIO>
opendkim-genkey -b 2048 -d <DOMINIO> -s $S && chown opendkim:opendkim $S.*
cp /etc/opendkim/KeyTable /etc/opendkim/KeyTable.bak-antes-$S
# 1. publique $S._domainkey.<DOMINIO> com o conteúdo de $S.txt e espere o dig responder
# 2. só então troque, na linha do domínio no KeyTable, o seletor e o arquivo:
#    default._domainkey.<DOMINIO> <DOMINIO>:<seletor>:/etc/opendkim/keys/<DOMINIO>/<seletor>.private
#    o primeiro campo (o nome da chave) fica igual: é ele que o SigningTable cita
systemctl reload opendkim
opendkim-testkey -d <DOMINIO> -s $S -k /etc/opendkim/keys/<DOMINIO>/$S.private -vvv
```

⚠️ Gere a chave nova, mas **não mexa no SigningTable nem no primeiro campo do
KeyTable**: o painel escreveu os dois casados, e descasá-los faz o Postfix mandar
sem assinatura, sem erro nenhum. E **publique o DNS antes de trocar o
KeyTable**: na ordem inversa, tudo que sair nesse meio-tempo vai com uma
assinatura que ninguém consegue conferir. O registro `default` antigo pode ficar
no DNS.

## Os quatro registros

Todos DNS-only. TXT e MX nunca passam por proxy, mas vale conferir.

⚠️ **Esta tabela é para domínio que não recebe e-mail em lugar nenhum.** Se a
medição 7 mostrou MX, SPF ou DMARC existentes, ela não se aplica como está: o MX
não entra, o SPF é o que já existe com o `ip4:`/`ip6:` somados, e o DMARC
existente fica. Ver `junto-com-google.md`.

Toda mudança de DNS é mostrada ao dono do domínio **antes**, como lista "hoje → depois"
de cada registro, e só é feita depois do sim dele. E-mail errado no DNS não dá
erro: o e-mail da empresa só para de chegar.

| Nome | Tipo | Valor |
|---|---|---|
| `<DOMINIO>` | TXT | `v=spf1 ip4:<IPV4> ip6:<IPV6> ~all` |
| `default._domainkey.<DOMINIO>` | TXT | `v=DKIM1; k=rsa; p=…` (o conteúdo de `default.txt`) |
| `_dmarc.<DOMINIO>` | TXT | `v=DMARC1; p=none; sp=none; adkim=r; aspf=r; rua=mailto:nao-responda@<DOMINIO>` |
| `<DOMINIO>` | MX 10 | `<HOSTNAME>` |

### ⚠️ O `ip6:` não é enfeite, e é o erro mais caro desta skill

O servidor tem IPv6, o Postfix **prefere** IPv6 quando o destino também tem, e o
Gmail tem. Com o SPF só em `ip4:`, a mensagem sai autenticada pela metade:

- o log do Postfix diz `status=sent (250)` — parece perfeito;
- o laudo do destinatário diz **`spf=softfail`**.

Medido: `softfail` com `ip4:` sozinho, e os três `pass` depois de acrescentar o
`ip6:`. Nada no servidor acusa a diferença, porque para ele o destino aceitou.

Rode `ip -6 addr` **antes** de escrever o SPF. Sempre.

### As outras três escolhas

- **SPF só com `ip4:` e `ip6:`, sem `a` e sem `mx`.** O SPF tem teto de **dez
  consultas de DNS** e estoura em `permerror` — e um `permerror` invalida o
  registro **inteiro**, então SPF passa a não valer nada. `ip4:` e `ip6:` custam
  zero consulta. Pelo mesmo motivo, `include:` entra com parcimônia.
- **`~all`, não `-all`.** Antes de mandar os provedores recusarem tudo que não
  sai do seu IP, você precisa ter certeza de que **nada mais** manda e-mail por
  esse domínio. Plataforma de marketing, ferramenta de disparo, formulário de
  site: qualquer um deles cai calado com `-all`. Apertar depois é trocar duas
  letras; descobrir que se fechou a porta da empresa é caro.
- **`p=none` com `rua=`.** O `p=none` só observa, e é o começo certo. O `rua=`
  é o que impede você de ficar cego: sem ele, quando a autenticação parar de
  funcionar, ninguém fica sabendo — a mensagem some e nenhum relatório volta.
  Com ele, Google e Microsoft mandam o diário do que passou e do que falhou, e
  é ele que diz quando dá para endurecer.
- ⚠️ **O MX aponta para `<HOSTNAME>`, nunca para um nome atrás de proxy.** Nome
  proxiado resolve para o IP do proxy, que não fala SMTP, e o e-mail de entrada
  morre calado. O hostname do servidor nunca é proxiado.

**Declare a consequência do MX:** com ele, o domínio passa a **receber** e-mail
nesse servidor. Ganha-se a devolução e os relatórios DMARC; em troca, mensagem
mandada a um endereço qualquer do domínio cai numa caixa que ninguém lê, em vez
de voltar ao remetente. Se isso não for desejado, o MX sai e o `rua=` sai junto.

⚠️ **Propagação engana.** Registro recém-criado pode responder vazio por dezenas
de segundos, **inclusive consultado no nameserver autoritativo**. Repita a
consulta um minuto depois antes de concluir que a criação falhou.

## O app

Conexão por **submission autenticada**, porta 587 com STARTTLS:

```
SMTP_HOST=<HOSTNAME>
SMTP_PORTA=587
SMTP_USUARIO=nao-responda@<DOMINIO>
SMTP_SENHA=…
EMAIL_REMETENTE=nao-responda@<DOMINIO>
EMAIL_REMETENTE_NOME="Nome Que Aparece"
```

⚠️ **O host é `<HOSTNAME>`, nunca `127.0.0.1`.** O certificado que o Postfix
apresenta é um Let's Encrypt real com **um SAN só**, o hostname. Apontando para
o loopback, o cliente recusa com `Hostname/IP does not match certificate's
altnames` — e o sintoma costuma ser a rota de login devolvendo 502. A saída é
usar o **nome**: o `/etc/hosts` resolve, e a conexão não sai da máquina.

⚠️ **Nunca desligue a verificação de certificado** (`rejectUnauthorized: false`,
`--insecure`, equivalentes). É a saída errada para o problema de cima, e ela
some do radar no dia em que o certificado vencer de verdade.

⚠️ **Remetente em dois campos, nome e endereço separados.** Um valor único no
formato `Nome <a@b>` tem espaço e `<`: sem aspas no `.env`, o `<` vira
redirecionamento e quebra **todo** comando de shell que carrega o arquivo,
apagando em silêncio as variáveis seguintes. A aplicação lê dos dois jeitos; só
o shell quebra. Dois campos eliminam o problema na origem.

Mais três, que valem em qualquer linguagem:

- **Um transporte para o processo inteiro, com pool.** Criar conexão por
  mensagem paga um aperto de mão SMTP por e-mail.
- **Tempo-limite explícito.** O padrão das bibliotecas costuma ser de minutos, e
  uma rota HTTP não pode ficar pendurada esperando um Postfix que não responde.
  Dez segundos para conectar, vinte para o soquete.
- **O que nunca entra no log:** a senha do SMTP, o link de redefinição e o
  código do segundo fator. O link **é** a credencial, e log costuma ser lido por
  muita gente. Logue o destinatário, que é o que permite achar o envio.

**Falha de envio deve subir**, não ser engolida. Quem decide o que fazer é quem
chamou: num link de redefinição dá para registrar e seguir, porque o token já
existe e dá para reenviar; num código de segundo fator a operação **tem** que
falhar, porque dizer "enviamos" sem ter enviado tranca a pessoa sem explicação.

## Provar que chegou

### Os três sinais, nesta ordem

```bash
grep -E 'nao-responda|dkim' /var/log/maillog | tail -5
mailq | tail -3
# se o maillog parou de ser escrito (medição 6), o mesmo pelo journal:
journalctl -u postfix -u opendkim --since '-15 min' --no-pager | grep -E 'sasl_username|DKIM|status='
```

1. `sasl_username=nao-responda@<DOMINIO>` — a caixa autenticou. Sem isso, a
   aplicação nem chegou ao Postfix.
2. `DKIM-Signature field added (s=default, d=<DOMINIO>)` — o OpenDKIM assinou.
3. `status=sent (250 …)` — o destino aceitou. `status=deferred` é o destino
   adiando; fila crescendo é a porta 25.

### A chave

```bash
opendkim-testkey -d <DOMINIO> -s default \
  -k /etc/opendkim/keys/<DOMINIO>/default.private -vvv
```

Espera-se `key OK`. O `key not secure` junto é só ausência de DNSSEC, e é
normal.

⚠️ **O `-k` é obrigatório.** Sem ele, o `opendkim-testkey` lê o `KeyFile` legado
do `opendkim.conf` — que o painel deixa para trás ao passar a usar `KeyTable` —
e reclama de um caminho que não existe, como se a chave estivesse errada.

### O laudo independente

O `250` não diz em que pasta a mensagem caiu. Peça um laudo de quem recebe:

```bash
# manda da própria caixa, autenticado, para o verificador
# a resposta chega no SEU servidor, porque o MX aponta para cá
D=/home/vmail/<DOMINIO>/nao-responda/Maildir/new
zcat "$D/$(ls -1t $D | head -1)" | grep -A4 'Summary of Results'
```

Destino: `check-auth@verifier.port25.com`. Ele responde com `SPF check`,
`DKIM check` e `iprev check` — os três precisam dizer `pass`.

⚠️ **O Dovecot guarda a mensagem comprimida.** Um `cat` devolve lixo binário e
parece mensagem corrompida. É `zcat`.

### E, uma vez, com os próprios olhos

Mande para uma conta real do provedor que os seus usuários usam e **veja em que
pasta caiu**. Se caiu no spam com os três `pass`, é reputação de IP novo, não
configuração: marcar "não é spam" nas primeiras vezes ensina o filtro, o volume
baixo ajuda, e o plano B continua sendo apontar as variáveis para um relay
autenticado — o que não muda uma linha de código.

## Auditar um servidor que já manda e-mail

Vale para qualquer domínio, inclusive os que você não montou:

```bash
# 1. o servidor tem IPv6? (se tiver, o SPF precisa do ip6:)
ssh <HOSTNAME> "ip -6 addr show scope global | grep -oP '(?<=inet6 )[0-9a-f:]+'"

# 2. o que o SPF diz hoje — procure ip6, e procure 'a' e 'mx' sobrando
dig +short TXT <DOMINIO> | grep spf1

# 3. o DMARC tem rua=? sem ele ninguém fica sabendo quando quebra
dig +short TXT _dmarc.<DOMINIO>

# 4. o MX aponta para nome proxiado?
dig +short MX <DOMINIO>

# 5. o domínio é local neste Postfix, mas o MX está em outro lugar?
ssh <HOSTNAME> "postmap -q <DOMINIO> proxy:mysql:/etc/postfix/mysql-virtual_domains.cf"

# 6. o mapa de certificados por nome está íntegro? (tem de mostrar 'Cipher is')
ssh <HOSTNAME> "openssl s_client -connect 127.0.0.1:25 -starttls smtp -servername mail.<DOMINIO> </dev/null 2>&1 | grep -E 'Cipher is|subject='"
```

Os três achados mais comuns, nesta ordem: SPF sem `ip6:` num servidor que tem
IPv6; DMARC `p=none` sem `rua=`; e SPF gastando consultas com `a` e `mx`. Domínio
sem SPF nenhum também aparece (domínio no Google em que ninguém publicou o
`include:_spf.google.com`): nesse caso o que o Google manda já sai sem SPF hoje.

## Erros comuns

| Sintoma | Causa |
|---|---|
| Laudo diz `spf=softfail`, mas o log diz `status=sent (250)` | SPF sem `ip6:` e o servidor saiu por IPv6. O mais comum de todos |
| A aplicação responde 502 no envio, log fala em `altnames` | `SMTP_HOST=127.0.0.1`. O certificado do Postfix é para o hostname |
| `. .env` no shell dá `syntax error` e as variáveis seguintes somem | Valor com espaço ou `<` sem aspas. A aplicação lê assim mesmo; o shell, não |
| Registro criado no DNS "não existe" no `dig` | Propagação. Espere um minuto e repita antes de recriar |
| Mexi no DNS pelo CyberPanel e nada mudou | A zona é servida por outro provedor. O PowerDNS local não é autoritativo |
| `opendkim-testkey` reclama de arquivo que não existe | Faltou o `-k`. Ele caiu no `KeyFile` legado do `opendkim.conf` |
| O laudo veio como lixo binário | O Dovecot guarda comprimido. É `zcat`, não `cat` |
| Aplicação manda, some, e `mailq` cresce | Porta 25 de saída fechada. Meça antes de depurar o resto |
| `dkim=none` no destino, e o Postfix não reclamou | OpenDKIM parado, ou `SigningTable` sem a linha do domínio |
| E-mail de entrada parou depois de mexer no DNS | Alguém ligou o proxy no nome para onde o MX aponta |
| Gmail "Enviar e-mail como" devolve `TLS Negotiation failed, the certificate doesn't match the host` depois de mudar o MX | Clientes configurados com `mail.<DOMINIO>` e o servidor só tem certificado do hostname. Emitir Let's Encrypt para `mail.<DOMINIO>` e entregar por SNI: linha no `/etc/postfix/vmail_ssl.map` + `postmap -F`, e `local_name` no `dovecot.conf` |
| E-mail do Gmail para TODOS os domínios do servidor parou de chegar logo depois de mexer no `vmail_ssl.map`; journal com `malformed BASE64 value` e `SSL_accept error` / `lost connection after STARTTLS` | O mapa foi refeito com `postmap` comum. Ele SEMPRE se refaz com `postmap -F hash:/etc/postfix/vmail_ssl.map` (o `-F` embute o certificado) e `postfix reload`. Depois testar: `openssl s_client -connect 127.0.0.1:25 -starttls smtp -servername <domínio>` tem de mostrar `Cipher is`. Num caso real, esquecer o `-F` deixou um servidor 4 h e meia sem receber do Google |
| Caixa recém-criada pelo `cyberpanel createEmail` recusa a senha certa (`535 authentication failed`, `doveadm auth test` = `auth failed`) | O CLI grava a senha em bcrypt (`{CRYPT}$2b$`) e o Dovecot do servidor não confere bcrypt. Regravar em `e_users` com `doveadm pw -s SHA512-CRYPT` trocando o prefixo para `{CRYPT}` e testar com `doveadm auth test`. Senha vinda de arquivo do Windows com `\r` também é recusada ("password having illegal chars") |
| Dovecot `failed` logo depois de criar um site ou uma caixa | O `createWebsite` pôs no `dovecot.conf` um `local_name <site>` com certificado que ainda não existe; o próximo restart (o `createEmail` faz um) derruba todas as caixas. Rodar `issueSSL` logo após o `createWebsite`, ou tirar o bloco, e sempre `doveconf -n` + `systemctl is-active dovecot` depois |
| Nenhuma caixa do servidor autentica, de uma hora para outra | O pacote `dovecot-mysql` sumiu numa atualização automática do sistema, e o Dovecot não consegue ler a tabela de caixas. Reinstalar o pacote. Num caso real, ficou dois meses sem ninguém notar |
| Outlook novo não envia: `Invalid authentication mechanism: 'LOGIN'` | O Dovecot só aceita `PLAIN`. Pôr `auth_mechanisms = plain login` num arquivo próprio em `conf.d` **com `!include_try` no `dovecot.conf`** |
| Mexi em `conf.d/10-auth.conf` (ou outro) e nada mudou | O `dovecot.conf` do CyberPanel **não carrega `conf.d/*`**, só o que tiver `!include_try` explícito no fim dele. Conferir sempre com `doveconf <opção>` |
| Depois de atualizar o CyberPanel, limites, LOGIN ou certificados de `mail.` voltaram ao que eram | O painel reescreve `dovecot.conf` e `dovecot-sql.conf.ext`, e os `!include_try` e o `LEFT JOIN` de cotas somem. Checklist em `caixas-para-pessoas.md` |
| Remetente recusa com `552 Caixa de e-mail cheia` | A caixa chegou no limite (cota do Dovecot, recusada já na entrada pelo `quota-status`). Ver `doveadm quota get -A`. É o comportamento certo; aumentar o limite é decisão do dono |
| Filtro, resposta automática ou Sieve: a entrega passa a dar `tempfail` | Caixa do CyberPanel não tem diretório home. O `sieve =` precisa de caminho absoluto |
| Os sites do servidor mandam para `alguem@<DOMINIO>`, que está no Google, e a mensagem volta `User unknown` ou some numa caixa daqui | O domínio virou **local** no Postfix ao ganhar a primeira caixa (`e_domains`), e o Postfix não procura mais o MX. Ver `junto-com-google.md` |
| Um e-mail redirecionado para o Gmail é recusado ou cai no spam | Redirecionamento sem SRS: o Gmail vê um remetente de fora saindo deste IP e o SPF falha. O servidor precisa do `postsrsd`. Ver `caixas-para-pessoas.md` |
| Desde que o servidor ganhou o `postsrsd`, o e-mail de um site novo sai com remetente `SRS0=…` e o DMARC falha | O domínio do site não está em `SRS_EXCLUDE_DOMAINS` (`/etc/default/postsrsd`). Acrescentar e `systemctl restart postsrsd` |
| Todo e-mail do servidor parou de sair por uns minutos, todos os domínios ao mesmo tempo, depois voltou sozinho | Limite da Hostinger: mais de 5 e-mails por minuto na porta 25 bloqueia o VPS. Freio por transporte próprio com `_destination_rate_delay`, ou relay autenticado para o volume |
| `cyberpanel issueSSL` deu certificado que o navegador recusa | Ele entregou um certificado de *staging*. Emitir com `acme.sh --issue -d <nome> -w /usr/local/lsws/Example/html --server letsencrypt` e `--install-cert` em `/etc/letsencrypt/live/<nome>/` |

## O que NÃO fazer

- Não prometer e-mail próprio antes de medir a porta 25 de saída.
- Não escrever o SPF antes de rodar `ip -6 addr`.
- Não usar `127.0.0.1` como host do SMTP, e **nunca** desligar a verificação de
  certificado para contornar isso.
- Não mandar a aplicação falar na porta 25 sem autenticação "porque é local":
  perde-se o remetente explícito e o dia em que o app sair da máquina.
- Não começar com `-all` nem com `p=reject`/`p=quarantine` sem um ciclo de
  observação.
- Não usar `a` nem `mx` no SPF.
- Não apontar o MX para um nome que possa estar atrás de proxy.
- Não gerar a chave DKIM por fora do painel.
- Não editar o DNS pelo CyberPanel quando a zona é de outro provedor.
- Não apagar a caixa nem trocar a senha dela sem atualizar o `.env` junto: onde
  o segundo fator depende do e-mail, isso deixa todo mundo de fora sem saída.
- Não concluir pelo `250` que a mensagem chegou à caixa de entrada.
- Não trocar o MX de um domínio que já recebe no Google ou em outro provedor, e
  não criar a primeira caixa dele no servidor sem ter lido `junto-com-google.md`.
- Não publicar um segundo SPF nem substituir o DMARC existente: soma-se ao que há.
- Não refazer o `vmail_ssl.map` sem `-F`: `postmap -F hash:/etc/postfix/vmail_ssl.map`.
- Não copiar as mensagens antigas de outro provedor sem pedido explícito:
  "migrar o e-mail" costuma querer dizer passar a receber e enviar no servidor
  novo, não trazer o histórico. Pergunte.
- Não apagar caixa (que apaga as mensagens junto) sem a confirmação do dono com
  o endereço nomeado.

O deploy da aplicação em si não é assunto desta skill.
