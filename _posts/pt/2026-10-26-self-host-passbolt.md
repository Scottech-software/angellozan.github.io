---
lang: pt
page_id: self_host_passbolt
permalink: /posts/self-host-passbolt
title: Hospedando o Passbolt como um Gerenciador de Senhas Privado
date: 2026-10-26
categories: [Docker, Security, Self-Hosting]
tags: [Tutorial, Self-Hosting, Passbolt, Docker, Tailscale, MariaDB, Password Manager]
author: Angel
description: Execute o Passbolt CE no Docker em um pequeno servidor doméstico, mantenha-o fora da internet aberta com o Tailscale e faça backups corretos, sem deixar o Docker mexer no firewall do host.
image:
  path: /assets/img/passbolt.jpg
  alt: Um cadeado sobre o teclado de um notebook
---

## Hospedando o Passbolt como um Gerenciador de Senhas Privado

Tirei minhas senhas, e as chaves de API que minhas próprias ferramentas usam, de um gerenciador de senhas pago na nuvem e as coloquei em um servidor que eu mesmo mantenho. O objetivo era simples: o cofre deve ficar em um hardware meu, nunca ser acessível pela internet aberta e ainda assim funcionar no meu notebook e no celular, onde quer que eu esteja.

O [Passbolt](https://www.passbolt.com/) se encaixa bem nisso:

- **Criptografia de ponta a ponta com OpenPGP.** Cada segredo é criptografado no navegador para cada usuário que pode lê-lo. O servidor guarda apenas texto cifrado e não consegue descriptografar nada.
- **Feito para compartilhar.** Pastas, grupos e permissões por item, o que também o torna um bom lugar para guardar credenciais de máquinas.
- **Tem uma CLI e uma API**, então servidores e scripts podem ler segredos sem intervenção humana.
- **A Community Edition é gratuita** e pode ser hospedada por você mesmo.

## Índice

- [O projeto](#o-projeto)
- [O que você precisa](#o-que-você-precisa)
- [Mantenha o Docker longe do firewall do host](#mantenha-o-docker-longe-do-firewall-do-host)
- [Estrutura de diretórios e segredos](#estrutura-de-diretórios-e-segredos)
- [Vincule o nginx ao localhost](#vincule-o-nginx-ao-localhost)
- [O arquivo compose](#o-arquivo-compose)
- [HTTPS e acesso remoto com Tailscale](#https-e-acesso-remoto-com-tailscale)
- [Crie o primeiro administrador](#crie-o-primeiro-administrador)
- [E-mail](#e-mail)
- [Backups](#backups)
- [Usuários de máquina](#usuários-de-máquina)
- [Ressalvas](#ressalvas)

---

## O projeto

```text
Notebook / celular (extensão do navegador ou app do Passbolt)
        │  Tailscale (WireGuard), somente tailnet
        ▼
tailscale serve  https://<server>.<tailnet>.ts.net:8443   (certificado real do Let's Encrypt)
        │
        ▼  127.0.0.1:4433
container passbolt (nginx + PHP)  ──>  container mariadb 127.0.0.1:3306
        │
        └── /srv/passbolt  (banco de dados, chave GPG do servidor, chaves JWT, segredos)
```

As escolhas que importam:

- **Nada escuta na LAN.** Os dois containers usam host networking, mas vinculam apenas a `127.0.0.1`. A única entrada é o Tailscale.
- **Nenhuma porta aberta no roteador.** O Tailscale funciona atrás de NAT, então o cofre fica invisível para a internet.
- **O Docker não gerencia o firewall.** O firewall do próprio host continua sendo a única fonte da verdade.
- **Docker em vez do pacote Debian.** A instalação por pacote do Passbolt espera um servidor limpo que ele possa configurar do zero. Minha máquina já roda outros serviços, então os containers mantêm o Passbolt isolado.

---

## O que você precisa

- Um pequeno servidor Debian ou Ubuntu sempre ligado. Um Raspberry Pi com 4 GB de RAM é suficiente; as imagens do Passbolt são construídas para arm64 e também para amd64.
- Uma conta no [Tailscale](https://tailscale.com/) com seu notebook e seu celular na tailnet, e **HTTPS Certificates** ativado na página de DNS do console de administração.
- Uma conta SMTP a partir da qual o Passbolt possa enviar e-mails (convites e recuperação de conta são enviados por e-mail).

---

## Mantenha o Docker longe do firewall do host

Por padrão, o Docker escreve suas próprias regras de iptables, e essas regras ignoram o ufw. Uma porta que você achava protegida pelo firewall pode acabar aberta para a LAN. Como todos os containers aqui usam host networking, o Docker não precisa mexer no firewall.

```bash
sudo apt install -y docker.io docker-compose zip   # zip é para o script de backup
```

`/etc/docker/daemon.json`:

```json
{
  "iptables": false,
  "ip6tables": false,
  "bridge": "none",
  "log-driver": "local",
  "log-opts": { "max-size": "10m", "max-file": "3" }
}
```

```bash
sudo systemctl restart docker
sudo iptables -S | grep -i docker   # não deve imprimir nada
ip link show docker0                # deve dizer que o dispositivo não existe
sudo ufw status verbose             # inalterado
```

O custo: a rede bridge padrão deixa de funcionar neste host. Todo container que você rodar aqui precisa de `network_mode: host`, então trate isso como uma regra para a máquina inteira. Os `log-opts` limitam os logs dos containers a 30 MB, o que importa em um cartão SD.

---

## Estrutura de diretórios e segredos

Tudo fica em `/srv/passbolt`, o que facilita backups e migrações.

```bash
sudo mkdir -p /srv/passbolt/{db,gpg,jwt,nginx,secrets}
cd /srv/passbolt

# Senhas como arquivos, nunca escritas direto no compose.yaml:
sudo sh -c 'openssl rand -base64 32 | tr -d "\n" > secrets/db_password'
sudo sh -c 'printf "%s" "you@example.com" > secrets/smtp_user'
sudo sh -c 'printf "%s" "<smtp-password>"  > secrets/smtp_password'

sudo chmod 700 secrets
sudo chmod 444 secrets/*        # legível dentro dos containers, a pasta em si fica restrita ao root

# Propriedade para os usuários dos containers:
sudo chown 2999:2999 db                           # MariaDB, veja a nota abaixo
sudo chown 33:33 gpg                              # www-data dentro da imagem do Passbolt
sudo chown root:33 jwt && sudo chmod 750 jwt      # o healthcheck exige exatamente essas permissões
```

**Por que o uid 2999 para o MariaDB?** A imagem do MariaDB roda como uid 999 por padrão. Com host networking e bind mounts, um uid que já pertence a um usuário do host significa que esse usuário é dono dos arquivos do seu banco de dados. Verifique `getent passwd 999` primeiro; eu tive um conflito e movi o MariaDB para um uid não utilizado.

---

## Vincule o nginx ao localhost

A imagem non-root do Passbolt roda o nginx nas portas `8080` (HTTP) e `4433` (HTTPS) em **todas** as interfaces. Com host networking, isso significa a LAN inteira. Copie os dois arquivos de configuração para fora da imagem, vincule-os a `127.0.0.1` e monte-os de volta.

```bash
cd /srv/passbolt
sudo docker create --name pb-tmp passbolt/passbolt:latest-ce-non-root
sudo docker cp pb-tmp:/etc/nginx/sites-available/nginx-passbolt.conf nginx/
sudo docker cp pb-tmp:/etc/nginx/snippets/passbolt-ssl.conf nginx/
sudo docker rm pb-tmp

# Remova os listeners IPv6 e vincule os IPv4 ao loopback:
sudo sed -i -E '/listen \[::\]:/d; s/listen 8080/listen 127.0.0.1:8080/' nginx/nginx-passbolt.conf
sudo sed -i -E '/listen \[::\]:/d; s/listen 4433/listen 127.0.0.1:4433/' nginx/passbolt-ssl.conf
grep -h listen nginx/*.conf     # toda linha deve dizer 127.0.0.1
```

O PHP-FPM dentro da imagem escuta em `127.0.0.1:9000`, que com host networking é a porta 9000 do host. Garanta que nada mais no host a use.

---

## O arquivo compose

`/srv/passbolt/compose.yaml`:

```yaml
services:
  db:
    image: mariadb:11.4
    restart: unless-stopped
    network_mode: host
    user: "2999:2999"
    command: ["--bind-address=127.0.0.1", "--port=3306", "--skip-name-resolve"]
    environment:
      MARIADB_RANDOM_ROOT_PASSWORD: "true"
      MARIADB_DATABASE: passbolt
      MARIADB_USER: passbolt
      MARIADB_PASSWORD_FILE: /run/secrets/db_password
    volumes:
      - ./db:/var/lib/mysql
      - ./secrets/db_password:/run/secrets/db_password:ro
    healthcheck:
      test: ["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]
      interval: 20s
      timeout: 5s
      retries: 10
      start_period: 30s

  passbolt:
    image: passbolt/passbolt:latest-ce-non-root
    restart: unless-stopped
    network_mode: host
    depends_on:
      db:
        condition: service_healthy
    environment:
      APP_FULL_BASE_URL: https://<server>.<tailnet>.ts.net:8443
      PASSBOLT_SSL_FORCE: "true"
      DATASOURCES_DEFAULT_HOST: 127.0.0.1
      DATASOURCES_DEFAULT_PORT: "3306"
      DATASOURCES_DEFAULT_DATABASE: passbolt
      DATASOURCES_DEFAULT_USERNAME: passbolt
      DATASOURCES_DEFAULT_PASSWORD_FILE: /run/secrets/db_password
      EMAIL_DEFAULT_FROM_NAME: Passbolt
      EMAIL_DEFAULT_FROM: you@example.com
      EMAIL_TRANSPORT_DEFAULT_HOST: smtp.example.com
      EMAIL_TRANSPORT_DEFAULT_PORT: "587"
      EMAIL_TRANSPORT_DEFAULT_TLS: "true"
      EMAIL_TRANSPORT_DEFAULT_USERNAME_FILE: /run/secrets/smtp_user
      EMAIL_TRANSPORT_DEFAULT_PASSWORD_FILE: /run/secrets/smtp_password
    extra_hosts:
      # Permite que o self-check do Passbolt alcance sua própria URL pública, veja abaixo:
      - "<server>.<tailnet>.ts.net:<server-tailscale-ip>"
    volumes:
      - ./gpg:/etc/passbolt/gpg
      - ./jwt:/etc/passbolt/jwt
      - ./nginx/nginx-passbolt.conf:/etc/nginx/sites-available/nginx-passbolt.conf:ro
      - ./nginx/passbolt-ssl.conf:/etc/nginx/snippets/passbolt-ssl.conf:ro
      - ./secrets/db_password:/run/secrets/db_password:ro
      - ./secrets/smtp_user:/run/secrets/smtp_user:ro
      - ./secrets/smtp_password:/run/secrets/smtp_password:ro
```

```bash
cd /srv/passbolt && sudo docker compose up -d
sudo docker compose logs -f passbolt   # a primeira inicialização gera a chave GPG do servidor e as chaves JWT
```

**Por que `extra_hosts`?** O healthcheck do Passbolt chama a própria `APP_FULL_BASE_URL`. Se o resolvedor de DNS do servidor não conhece os nomes MagicDNS do Tailscale, essa chamada falha e o healthcheck reclama. Mapear o nome para o IP Tailscale do servidor dentro do container resolve isso sem mexer no DNS do host.

---

## HTTPS e acesso remoto com Tailscale

O `tailscale serve` encerra o TLS com um certificado real para o nome do servidor na tailnet e faz proxy para o container no loopback. Somente dispositivos da sua tailnet conseguem acessá-lo.

```bash
sudo tailscale serve --bg --https=8443 https+insecure://127.0.0.1:4433
sudo tailscale serve status
```

`https+insecure` não é problema aqui. O trecho que ele cobre nunca sai da máquina, e o certificado do próprio container é autoassinado. Fazer proxy para a porta HTTPS em vez de `8080` evita um loop de redirecionamento, porque o `PASSBOLT_SSL_FORCE` envia HTTP simples para HTTPS.

Para tirar o cofre do ar instantaneamente: `sudo tailscale serve --https=8443 off`.

---

## Crie o primeiro administrador

**Pegadinha:** o entrypoint da imagem transforma as variáveis `*_FILE` em variáveis de ambiente reais apenas para o seu próprio processo. Um shell aberto com `docker compose exec` não as recebe, então a CLI do Passbolt não consegue se conectar ao banco de dados. Exporte-as primeiro:

```bash
pb() {
  sudo docker compose -f /srv/passbolt/compose.yaml exec passbolt bash -c '
    export DATASOURCES_DEFAULT_PASSWORD="$(cat "$DATASOURCES_DEFAULT_PASSWORD_FILE")"
    export EMAIL_TRANSPORT_DEFAULT_USERNAME="$(cat "$EMAIL_TRANSPORT_DEFAULT_USERNAME_FILE")"
    export EMAIL_TRANSPORT_DEFAULT_PASSWORD="$(cat "$EMAIL_TRANSPORT_DEFAULT_PASSWORD_FILE")"
    /usr/share/php/passbolt/bin/cake passbolt "$@"' _ "$@"
}

pb healthcheck            # o objetivo é "[PASS] No error found"
pb register_user -u you@example.com -f First -l Last -r admin
```

O `register_user` imprime um link de configuração. Abra-o em um dispositivo da tailnet com a extensão do Passbolt instalada no navegador, gere sua chave e **guarde o kit de recuperação offline**. Sem ele, perder o perfil do navegador significa perder o cofre.

Depois, instale o app para celular e faça o pareamento a partir da extensão.

---

## E-mail

O Passbolt envia convites, e-mails de recuperação e notificações de compartilhamento. A maioria dos provedores SMTP hospedados funciona direto com as configurações do arquivo compose.

Se você usa um relay local (bridge) com certificado autoassinado, fixe esse certificado em vez de desativar totalmente a verificação:

```yaml
      PASSBOLT_PLUGINS_SMTP_SETTINGS_SECURITY_SSL_CAFILE: /etc/ssl/smtp-relay.pem
      PASSBOLT_PLUGINS_SMTP_SETTINGS_SECURITY_SSL_VERIFY_PEER_NAME: "false"
```

e monte o certificado como somente leitura nesse caminho. Envie um teste em **Administration > Email server**.

---

## Backups

Um backup do Passbolt precisa incluir mais do que o banco de dados. A **chave GPG do servidor** e as **chaves JWT** são tão importantes quanto: sem a chave do servidor, um banco de dados restaurado é inútil.

`/usr/local/sbin/passbolt-backup`:

```bash
#!/bin/sh
set -eu
umask 077
DEST=/var/backups/passbolt
STAMP=$(date +%Y%m%d-%H%M)
WORK=$(mktemp -d)
trap 'rm -rf "$WORK"' EXIT

mkdir -p "$DEST"
cd /srv/passbolt

docker compose exec -T db sh -c \
  'MYSQL_PWD="$(cat /run/secrets/db_password)" mariadb-dump -u passbolt --single-transaction passbolt' \
  > "$WORK/passbolt.sql"

cp -a gpg jwt nginx compose.yaml secrets/db_password "$WORK/"
(cd "$WORK" && zip -qr "$DEST/passbolt-$STAMP.zip" .)

find "$DEST" -name 'passbolt-*.zip' -mtime +28 -delete
```

```bash
sudo chmod 700 /usr/local/sbin/passbolt-backup
echo '45 3 * * * root /usr/local/sbin/passbolt-backup' | sudo tee /etc/cron.d/passbolt-backup
```

Depois, leve os zips **para fora da máquina**. Eu os puxo toda noite para outra máquina em um disco criptografado, e uma verificação me avisa se o backup mais recente tiver mais de dois dias. Um backup que existe apenas no servidor que ele protege não é um backup.

Teste uma restauração pelo menos uma vez: diretório novo, descompactar, importar o `passbolt.sql`, iniciar os containers, fazer login.

---

## Usuários de máquina

Também uso o Passbolt como fonte única da verdade para as chaves de API de que meus servidores e scripts precisam. Cada máquina recebe seu próprio usuário no Passbolt e lê os segredos com o [go-passbolt-cli](https://github.com/passbolt/go-passbolt-cli).

Uma coisa que me surpreendeu: **segredos criados por um usuário de máquina ficam invisíveis para todos os outros** até serem compartilhados item por item. É a criptografia de ponta a ponta fazendo seu trabalho: o servidor não consegue recriptografar nada em seu nome. Planeje ao contrário. Crie os itens como você mesmo, em uma pasta, e dê ao usuário de máquina acesso somente leitura a essa pasta.

---

## Ressalvas

- **Suas senhas só ficam acessíveis enquanto o servidor e o Tailscale estiverem no ar.** Guarde uma cópia impressa ou offline das poucas credenciais de que você precisaria para consertar o próprio servidor.
- **Fixe as versões das imagens** quando tudo estiver funcionando e leia as notas de lançamento antes de atualizá-las. `latest-ce-non-root` é conveniente para a primeira instalação, não para atualizações automáticas.
- **Proteja o kit de recuperação e a chave GPG do servidor** tanto quanto o próprio cofre. Elas são as chaves de tudo.

Desde a mudança, cancelei a assinatura da nuvem, e os segredos que minhas ferramentas usam são sincronizados a partir de um único lugar que eu controlo.
