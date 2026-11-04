# Caixas para pessoas: webmail, celular, Outlook

Complemento de `SKILL.md` para quando as caixas são de gente (setor, funcionário),
não de sistema. Pressupõe as sete medições e o DNS já feitos. Se o domínio recebe
no Google, leia `junto-com-google.md` antes deste.

O que muda em relação à caixa de sistema: a pessoa entra pelo navegador, pelo
celular e pelo Outlook; esquece a senha; enche a caixa; pede redirecionamento e
resposta automática. Cada uma dessas coisas tem uma armadilha no CyberPanel.

## Antes de criar: o que perguntar ao dono

- Quais endereços, para quem, e o limite de cada caixa. Toda caixa tem limite,
  inclusive as criadas depois: sem limite, uma caixa enche o disco do servidor
  e derruba os sites junto. Um padrão razoável é 1 GB.
- Se alguma caixa redireciona uma cópia para outro e-mail.
- Se vai ter histórico de outro provedor a trazer. Por padrão **não**: só com
  pedido explícito.

## Escolher o servidor

O servidor que já tem e-mail de pessoas funcionando e provado é o candidato
natural, porque o webmail, o certificado de `mail.` e os ajustes de Dovecot já
estão lá, e um único Roundcube pode atender vários domínios. Um servidor que
nunca recebeu caixa de gente começa da seção "Webmail do zero" e de todos os
itens abaixo.

Rode as medições no servidor escolhido. Servidor cujo Postfix é recusado pelo
Gmail não
recebe caixa de pessoa antes de achar o motivo: PTR, listas de bloqueio ou porta 25.

## Criar a caixa, e conferir que a senha funciona

```bash
cyberpanel createEmail --domainName <DOMINIO> --userName <usuario> --password '<senha>'
doveadm auth test <usuario>@<DOMINIO> '<senha>'
```

⚠️ Em alguns servidores o CLI grava a senha em bcrypt (`{CRYPT}$2b$`) e o Dovecot
não confere bcrypt: o `doveadm auth test` diz `auth failed` com a senha certa.
Regrave em SHA512-CRYPT, mantendo o prefixo `{CRYPT}`:

```bash
H=$(doveadm pw -s SHA512-CRYPT -p '<senha>' | sed 's/^{SHA512-CRYPT}/{CRYPT}/')
mysql cyberpanel -e "UPDATE e_users SET password='$H' WHERE email='<usuario>@<DOMINIO>'"
doveadm auth cache flush; doveadm auth test <usuario>@<DOMINIO> '<senha>'
```

Senha lida de arquivo do Windows chega com `\r` no fim e é recusada ("password
having illegal chars"). Limpe com `tr -d '\r'`.

A senha da pessoa vai para ela por um canal só, nunca para chat, para o git
nem para o log.

## Limite de cada caixa

O desenho que funciona no CyberPanel: uma regra padrão para todas as caixas e
uma tabela própria para as exceções.

```bash
# 1. regra padrão, em arquivo próprio (o dovecot.conf do painel só lê conf.d com include explícito)
cat > /etc/dovecot/conf.d/96-cotas.conf <<'EOF'
plugin {
  quota = maildir:User quota
  quota_rule = *:storage=1G
  quota_status_success = DUNNO
  quota_status_nouser = DUNNO
  quota_status_overquota = "552 5.2.2 Caixa de e-mail cheia"
}
EOF
echo '!include_try conf.d/96-cotas.conf' >> /etc/dovecot/dovecot.conf
# (o plugin quota precisa estar em mail_plugins, e o serviço quota-status
#  escutando, por exemplo, em inet 127.0.0.1:12340)

# 2. exceções: tabela no banco do painel
mysql cyberpanel -e "CREATE TABLE IF NOT EXISTS cotas_email (email VARCHAR(191) PRIMARY KEY, bytes BIGINT NOT NULL, origem VARCHAR(50))"
# e no user_query de /etc/dovecot/dovecot-sql.conf.ext:
#   SELECT '5000' AS uid, '5000' AS gid, u.mail, CONCAT('*:bytes=', c.bytes) AS quota_rule
#   FROM e_users u LEFT JOIN cotas_email c ON c.email = u.email WHERE u.email='%u'

# 3. recusar na entrada: no main.cf, ao fim de smtpd_recipient_restrictions,
#    check_policy_service inet:127.0.0.1:12340
```

Confira `mail_plugins`, os nomes de serviço e a versão do Dovecot do seu
servidor antes de colar: é um desenho, não um arquivo pronto. Mudar um limite:

```bash
mysql cyberpanel -e "INSERT INTO cotas_email (email, bytes, origem) VALUES ('<caixa>', <bytes>, 'pedido') ON DUPLICATE KEY UPDATE bytes=VALUES(bytes)"
doveadm auth cache flush; doveadm quota get -u <caixa>
```

Vale acrescentar um aviso ao dono da caixa em 80% e 95% (`quota_warning` do
Dovecot chamando um script que manda e-mail).

## Celular e Outlook: o nome do servidor

Duas opções, nesta ordem de preferência:

1. **O próprio `<HOSTNAME>`** como servidor de entrada e de saída. Já tem
   certificado válido e não exige nada novo. Funciona em qualquer aplicativo.
2. **`mail.<DOMINIO>`**, quando o cliente quer o nome da empresa ou já tem
   aparelhos configurados assim (caso de quem veio de outro provedor). Exige
   certificado próprio entregue por SNI:
   - DNS: `mail` tipo A para o IPv4 do servidor, **sem proxy** (e AAAA para o IPv6).
   - O nome entra no `map` do vhost do webmail em `httpd_config.conf` (em
     **todos** os listeners onde o vhost aparece), para o desafio HTTP passar.
   - Certificado: `acme.sh --issue -d mail.<DOMINIO> -w /usr/local/lsws/Example/html --server letsencrypt`,
     instalado em `/etc/letsencrypt/live/mail.<DOMINIO>/` com `--reloadcmd`
     que recarregue Postfix e Dovecot.
   - Postfix: linha `mail.<DOMINIO> <privkey.pem> <fullchain.pem>` em
     `/etc/postfix/vmail_ssl.map`, depois **`postmap -F hash:/etc/postfix/vmail_ssl.map`**
     e `postfix reload`. ⚠️ Sem o `-F` todo remetente que manda SNI, o Google
     inclusive, cai no STARTTLS e para de entregar em **todos** os domínios do
     servidor (num caso real, 4 h e meia sem receber do Gmail).
   - Dovecot: bloco `local_name mail.<DOMINIO> { ssl_cert = <…; ssl_key = <… }`
     no `dovecot.conf`, `doveconf -n`, `systemctl reload dovecot`.
   - Teste: `openssl s_client -connect 127.0.0.1:25 -starttls smtp -servername mail.<DOMINIO>`
     tem de mostrar `Cipher is` e o nome certo no certificado; o mesmo na 993.

Dados para a pessoa: IMAP 993 SSL; SMTP 465 SSL ou 587 STARTTLS; usuário é o
endereço completo.

O Outlook novo autentica com `LOGIN`. O Dovecot do CyberPanel só aceita `PLAIN`
de fábrica: `auth_mechanisms = plain login` em arquivo próprio em `conf.d`, com
`!include_try` no fim do `dovecot.conf`, e conferir com `doveconf auth_mechanisms`.

## Webmail

### Servidor que já tem um Roundcube servindo vários domínios

Um só Roundcube, num vhost `webmail.<DOMINIO-PRINCIPAL>`, pode servir todos os
domínios do servidor. Para um domínio novo:

1. Acrescentar `webmail.<DOMINIO>` e `mail.<DOMINIO>` à linha `map` desse vhost
   em `/usr/local/lsws/conf/httpd_config.conf`. A linha aparece em **mais de um
   listener**; todas recebem os nomes.
2. Reemitir o certificado compartilhado do webmail com o nome novo junto dos que
   ele já tem (veja como foi emitido em `/root/.acme.sh` antes de mexer).
3. DNS `webmail` para o servidor. Se a zona do Cloudflare estiver em SSL
   *flexible*, uma Configuration Rule põe o `webmail` em *Full (strict)*: em
   *flexible* a senha trafega em HTTP entre o Cloudflare e o servidor.
4. Login: com `username_domain` apontando para o domínio principal, quem
   é de outro domínio entra com o **endereço completo**. Diga isso à pessoa.

### Webmail do zero (servidor sem Roundcube)

O CyberPanel 3.0 não traz webmail que funcione. O padrão que deu certo:

- Site `webmail.<DOMINIO>` no painel; Roundcube em
  `/home/webmail.<DOMINIO>/roundcube`; banco SQLite em `/home/webmail.<DOMINIO>/db`.
- ⚠️ **A raiz do vhost é `roundcube/public_html`**, não a pasta do Roundcube. O
  OpenLiteSpeed não aplica os bloqueios do `.htaccess`, e na raiz errada a
  configuração e o banco ficam baixáveis por qualquer um.
- `imap_host = ssl://<HOSTNAME>:993` e `smtp_host = tls://<HOSTNAME>:587`. Nunca
  `127.0.0.1`: o certificado é do hostname.
- `username_domain = <DOMINIO>` deixa entrar sem digitar o `@`.
- Tema com a marca do cliente: herdar o Elastic (`skins/<nome>`), cores em
  `.less` recompiladas com `lessc`. O Elastic procura `styles.css` sem `.min`, e
  as duas cópias precisam existir.
- Atualizar: trocar a pasta mantendo `config/config.inc.php` e rodar `bin/update.sh`.
- Abrir a caixa sem senha a partir de um sistema próprio (tela de
  administração): usuário mestre no Dovecot (`/etc/dovecot/master-users`, login
  `<caixa>*<mestre>`) e um plugin no Roundcube que troca um token de uso único,
  de poucos segundos, gerado pelo sistema. Quem tem esse acesso lê qualquer
  caixa: só administrador, e cada abertura registrada.

## Redirecionar uma cópia para outro e-mail

No CyberPanel o redirecionamento é linha em `cyberpanel.e_forwardings`. Para
manter a cópia na caixa, são duas linhas: uma com destino igual à origem e outra
para o destino externo.

⚠️ Redirecionar para o Gmail exige **SRS** (`postsrsd`, ligado no Postfix por
`sender_canonical_maps` e `recipient_canonical_maps`). Sem ele o Gmail vê um
remetente de fora saindo deste IP, o SPF falha e a mensagem é recusada ou vai
para o spam. Com ele, todo domínio que **manda** e-mail deste servidor (sites,
sistemas) entra em `SRS_EXCLUDE_DOMAINS` em `/etc/default/postsrsd`, senão o
remetente dos sistemas é reescrito e o DMARC deles falha. Domínio novo no
servidor: acrescentar à lista e `systemctl restart postsrsd`.

## Backup

`/home/vmail/<DOMINIO>` entra no backup do servidor no mesmo dia em que a
primeira caixa de pessoa é criada. Teste uma restauração antes de dizer que há backup.

## Trazer o histórico de outro provedor (só a pedido)

- Senha igual à de antes: o hash do `shadow` do cPanel (`{CRYPT}$1$…`) pode ir
  direto para `e_users`; a pessoa não percebe a troca.
- `rsync -a` como root traz o uid do outro servidor e, com hardlink, quebra a
  caixa real ("Permission denied" no `dovecot-uidlist`). Use
  `rsync -rlt --chown=vmail:vmail`.
- `doveadm backup` falha com "INBOX can't be deleted"; `doveadm import` cria
  pastas sem o prefixo `INBOX.`: use `INBOX` como pasta-mãe.
- Hospedagem compartilhada (HostGator e afins) bloqueia o IP por uns minutos
  com muitas conexões SSH seguidas:
  uma conexão só, com `ControlMaster`.
- Antes de virar o MX, guarde uma cópia do DNS antigo inteiro.

## Depois de atualizar o CyberPanel

O painel reescreve arquivos que os ajustes acima mudaram. Confira:

```bash
grep -n '!include_try' /etc/dovecot/dovecot.conf          # cotas, LOGIN e o que mais houver
grep -c cotas_email /etc/dovecot/dovecot-sql.conf.ext      # 1 = limites próprios ainda valem
doveconf auth_mechanisms                                   # plain login
rpm -q dovecot-mysql                                       # instalado
openssl s_client -connect 127.0.0.1:25 -starttls smtp -servername mail.<DOMINIO> </dev/null 2>&1 | grep 'Cipher is'
systemctl is-active postfix dovecot opendkim postsrsd
```

Se o `dovecot-sql.conf.ext` foi reescrito, as caixas com limite próprio caem no
padrão de 1 GB sem aviso.

## Apagar uma caixa

O CyberPanel apaga só a linha da tabela; as mensagens ficam em
`/home/vmail/<DOMINIO>/<usuario>` e uma caixa recriada com o mesmo nome herda
tudo, inclusive as mensagens de outra pessoa. Apagar a caixa é apagar a pasta
também, e isso não tem volta: confirmação do dono com o endereço nomeado.

## Prova de que a caixa está pronta para entregar

1. `doveadm auth test` com a senha que a pessoa vai usar.
2. Entrar no webmail com ela.
3. Mandar dela para uma conta Gmail e para `check-auth@verifier.port25.com`:
   SPF, DKIM e iprev em `pass`.
4. Mandar de uma conta Gmail para ela e ver chegar.
5. Se tiver `mail.<DOMINIO>`, o `openssl s_client` da seção acima.
