# Skill: e-mail próprio no VPS com CyberPanel

Skill para o [Claude Code](https://claude.com/claude-code) montar, auditar e
consertar e-mail no Postfix/Dovecot de um VPS com CyberPanel (Hostinger,
AlmaLinux, OpenLiteSpeed).

Cobre três trabalhos:

| Trabalho | Arquivo |
|---|---|
| Um sistema precisa **mandar** e-mail (redefinição de senha, código de login, aviso, boleto) | `SKILL.md` |
| **Caixas para pessoas** com webmail (Roundcube), celular, Outlook, limite por caixa e redirecionamento | `caixas-para-pessoas.md` |
| Domínio que **já recebe no Google Workspace** e quer endereços extras no servidor (entrega dividida) | `junto-com-google.md` |

E o diagnóstico de quando "foi enviado" e não chegou: SPF em `softfail` por
falta do `ip6:`, porta 25 fechada, limite de 5 e-mails por minuto da Hostinger,
certificado por nome (SNI) quebrado por `postmap` sem `-F`, senha certa
recusada pelo Dovecot, Outlook que não autentica, caixa cheia e mais.

Tudo saiu de servidores em produção, inclusive a parte de Google Workspace
(`junto-com-google.md`), validada em novembro de 2026.

## Instalar

Copie a pasta `configurar-email-no-vps` para a pasta de skills do Claude Code:

```bash
# para todos os projetos
git clone https://github.com/tpwebmaster/skill-configurar-email-no-vps.git
cp -r skill-configurar-email-no-vps/configurar-email-no-vps ~/.claude/skills/

# ou só para um projeto
cp -r skill-configurar-email-no-vps/configurar-email-no-vps <projeto>/.claude/skills/
```

No Windows, `~/.claude/skills` é `C:\Users\<você>\.claude\skills`.

Abra uma sessão nova do Claude Code. A skill entra sozinha quando o assunto é
e-mail no servidor ("criar uma caixa", "o e-mail não está chegando", "webmail",
"SPF") ou quando você pede `/configurar-email-no-vps`.

## Como ela trabalha

- Mede antes de mexer: porta 25, IPv4 e IPv6, PTR, listas de bloqueio, quem
  responde pelo DNS e o que o domínio já tem de MX, SPF, DKIM e DMARC.
- Não troca MX de domínio que já recebe em outro lugar e soma ao SPF e ao DMARC
  existentes, em vez de substituir.
- Mostra toda mudança de DNS como "hoje → depois" e espera a sua confirmação.
- Só dá como pronto com prova de quem recebe (laudo SPF/DKIM/iprev), não com o
  `250` do log.

Nos arquivos, `<DOMINIO>` é o domínio do e-mail e `<HOSTNAME>` é o nome do
servidor (`hostname -f`).

## Licença

MIT. Veja `LICENSE`.
