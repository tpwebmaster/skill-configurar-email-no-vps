# Domínio no Google Workspace com caixas extras no servidor

Caso: o domínio já recebe no Google (MX `aspmx.l.google.com`), as pessoas que
estão lá continuam lá, e o cliente quer **outros** endereços do mesmo domínio
no servidor, com webmail, sem pagar licença do Google para cada um.

⚠️ Este desenho foi escrito a partir da documentação do Google e do
comportamento do Postfix, e ainda não tinha rodado em produção quando foi
publicado. Na primeira vez, siga os testes do passo 4 à risca e corrija este
arquivo com o que divergir.

Leia inteiro antes de criar a primeira caixa. O passo que parece inofensivo,
`cyberpanel createEmail`, é o que quebra: a partir dele o Postfix do servidor
considera o domínio **dele**.

## Por que é diferente

1. **O MX continua no Google.** Todo e-mail de fora chega primeiro no Google.
   É o Google que repassa ao servidor o que for para um endereço que ele não
   conhece (o recurso se chama *split delivery*, entrega dividida).
2. **O servidor passa a achar que o domínio é local.** Com a primeira caixa, o
   domínio entra em `e_domains` e o Postfix deixa de consultar o MX para ele.
   Daí em diante, todo e-mail que **sai deste servidor** para `fulano@<DOMINIO>`
   que mora no Google (formulário de site, aviso de sistema, código de login,
   resposta de quem tem caixa aqui) volta com `User unknown`, ou some numa caixa
   local que ninguém lê. Já aconteceu num caso real, e ninguém percebeu por dias.

O desenho só fica certo com as duas metades montadas: a rota no Google e a lista
dos endereços do Google no servidor.

## Decisão: mesmo domínio ou subdomínio

| | Mesmo domínio (`setor@<DOMINIO>`) | Subdomínio (`setor@equipe.<DOMINIO>`) |
|---|---|---|
| Endereço | o que o cliente quer | diferente do resto da empresa |
| Precisa do admin do Google | sim, para criar a rota | não |
| Manutenção | lista no servidor com **todos** os endereços do Google (pessoas, grupos e apelidos), atualizada a cada usuário criado ou removido no Google | nenhuma |
| Risco | lista desatualizada = e-mail entre os dois lados se perde ou faz laço | nenhum para o Google |
| DNS | soma no SPF, DKIM com seletor próprio, MX intocado | MX, SPF, DKIM e DMARC próprios do subdomínio, os quatro registros de `SKILL.md` |

O padrão é o mesmo domínio, porque é o que o cliente pede. O subdomínio é a
resposta quando ninguém tem acesso de administrador ao Google, ou quando os
usuários do Google mudam toda semana e ninguém vai manter a lista. A escolha é
do dono do domínio: apresente a tabela.

## Antes de começar

- Medições de `SKILL.md`, com atenção à 7: anote o MX, o SPF, o DMARC e os
  seletores DKIM que já existem. Eles ficam.
- A lista completa de endereços que vivem no Google: usuários, **grupos**
  (`contato@`, `financeiro@` costumam ser grupos) e apelidos. Vem do admin do
  Google (exportação de usuários e de grupos). Sem ela não comece.
- Os sites e sistemas **deste servidor** que mandam e-mail para o domínio. São
  eles que quebram se a lista estiver incompleta.
- Nenhum endereço pode existir nos dois lados. Se existir no Google, o Google
  entrega lá e o servidor nunca recebe.
- Quem é o administrador do Google. Se for o cliente, a rota é ensinada a ele
  passo a passo, não feita por nós.

## 1. DNS (mostrar "hoje → depois" e esperar o sim)

| Registro | Hoje | Depois |
|---|---|---|
| MX | Google | **igual** |
| SPF | `v=spf1 include:_spf.google.com ~all`, ou nada | `v=spf1 include:_spf.google.com ip4:<IPV4> ip6:<IPV6> ~all` (um registro só) |
| DKIM do Google | `google._domainkey` | **igual** |
| DKIM do servidor | — | `<seletor>._domainkey`, seletor próprio (ver `SKILL.md`); nunca `google` |
| DMARC | o que existir | **igual**. Se não existir: `p=none` com `rua=` para uma caixa que alguém lê |

Domínio no Google sem SPF nenhum já hoje manda sem SPF: o registro novo conserta
o Google também, por isso o `include:_spf.google.com` vai junto.

## 2. Servidor: a lista dos endereços do Google

Crie as caixas do servidor normalmente (`caixas-para-pessoas.md`). Na mesma
sessão, antes de dar por pronto, ensine ao Postfix que os endereços do Google
saem para o Google. Confira primeiro o que já existe (`postconf -n | grep -E 'transport_maps|virtual_mailbox_maps'`)
para somar, não substituir.

```bash
D=<DOMINIO>
# um endereço do Google por linha, pessoas + grupos + apelidos
cat > /etc/postfix/google_$D.txt <<'EOF'
fulano@<DOMINIO>
contato@<DOMINIO>
EOF

# rota: esses endereços vão para o MX do Google
awk '{print $1" smtp:[aspmx.l.google.com]:25"}' /etc/postfix/google_$D.txt > /etc/postfix/transport_google
# aceitação: sem isto o Postfix recusa na porta 587 com "User unknown" antes de olhar a rota
awk '{print $1" google"}' /etc/postfix/google_$D.txt > /etc/postfix/caixas_google
postmap /etc/postfix/transport_google /etc/postfix/caixas_google

cp /etc/postfix/main.cf /etc/postfix/main.cf.bak-antes-google-$D
postconf -e "transport_maps = hash:/etc/postfix/transport_google"
postconf -e "virtual_mailbox_maps = $(postconf -h virtual_mailbox_maps), hash:/etc/postfix/caixas_google"
postfix check && postfix reload
```

Com mais de um domínio nessa situação no mesmo servidor, os arquivos
`transport_google` e `caixas_google` recebem as linhas de todos.

⚠️ **Laço.** Endereço que está na lista mas não existe mais no Google: o Google
devolve para o servidor (não reconhece), o servidor manda para o Google (está
na lista), e assim até o Postfix desistir. Usuário removido no Google sai da
lista no mesmo dia. Usuário novo no Google entra na lista no mesmo dia, senão o
que sai deste servidor para ele volta `User unknown`.

⚠️ O CyberPanel não conhece esses dois arquivos. Depois de atualizar o painel,
confira que `transport_maps` e `virtual_mailbox_maps` ainda os têm.

## 3. Google: a rota (Admin console)

1. **Apps > Google Workspace > Gmail > Hosts**: adicionar rota com o host
   `<HOSTNAME>`, porta 25. Marcar exigir TLS e validar o certificado (o do
   hostname é Let's Encrypt válido). Nunca um nome atrás de proxy.
2. **Apps > Google Workspace > Gmail > Routing**, *Configure* ou *Add another rule*:
   - nome da regra (sem nome não salva);
   - marcar **Inbound** e **Internal inbound** (sem o segundo, quem está no
     Google não consegue escrever para quem está no servidor);
   - *Modify message > Change route*, escolher a rota do passo 1;
   - em *Show more options*, marcar **só** *All inactive and unrecognized
     accounts*. Deixar desmarcados *Active user account* e *Group account*.
3. Salvar. O Google avisa que leva até 24 h; costuma ser bem menos.

Os nomes dos menus mudam com o tempo (já foi *Default routing*). Referência:
[split delivery no suporte do Google](https://knowledge.workspace.google.com/admin/gmail/advanced/send-email-to-2-email-systems-with-split-delivery).

## 4. Provar os seis caminhos

Só está pronto quando os seis funcionarem. Use uma caixa de teste do servidor e
uma pessoa real do Google (com o sim dela).

| # | De → para | Onde olhar |
|---|---|---|
| A | Gmail de fora → pessoa do Google | chega no Gmail dela, como sempre |
| B | Gmail de fora → caixa do servidor | chega no webmail. No servidor, `journalctl -u postfix` mostra a entrega vindo de um IP do Google |
| C | Caixa do servidor → pessoa do Google | chega no Gmail dela; no servidor, `relay=aspmx.l.google.com` e `status=sent` |
| D | Pessoa do Google → caixa do servidor | chega no webmail (é o *Internal inbound*) |
| E | Site ou sistema deste servidor → pessoa do Google | chega (formulário, código de login) |
| F | Caixa do servidor → `check-auth@verifier.port25.com` | SPF, DKIM e iprev em `pass` |

Mande também para um endereço que não existe em lado nenhum: tem de voltar
recusado em poucos minutos, não ficar circulando.

## Erros deste caso

| Sintoma | Causa |
|---|---|
| E-mail de fora para a caixa do servidor não chega, e o Google não devolve | Rota ainda propagando (até 24 h), regra sem *unrecognized accounts*, ou o endereço também existe no Google |
| Quem está no Google escreve para a caixa do servidor e recebe "endereço não encontrado" | Falta *Internal inbound* na regra |
| Formulário do site deste servidor para `contato@<DOMINIO>` (grupo do Google) parou | `contato@` não está na lista do passo 2. Grupos e apelidos também entram |
| Mensagem com dezenas de `Received:` e devolução por *mail loop* | Endereço na lista do servidor que não existe mais no Google |
| SPF `fail` para o que o Google manda depois da mudança | O SPF foi substituído em vez de somado: o `include:_spf.google.com` sumiu |
