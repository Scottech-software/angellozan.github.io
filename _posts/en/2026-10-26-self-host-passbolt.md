---
lang: en
page_id: self_host_passbolt
permalink: /posts/self-host-passbolt
title: Self-Hosting Passbolt as a Private Password Manager
date: 2026-10-26
categories: [Docker, Security, Self-Hosting]
tags: [Tutorial, Self-Hosting, Passbolt, Docker, Tailscale, MariaDB, Password Manager]
author: Angel
description: Run Passbolt CE in Docker on a small home server, keep it off the open internet with Tailscale, and back it up properly, without letting Docker touch the host firewall.
image:
  path: /assets/img/passbolt.jpg
  alt: A padlock resting on a laptop keyboard
---

## Self-Hosting Passbolt as a Private Password Manager

I moved my passwords, and the API keys my own tooling uses, off a paid cloud password manager and onto a server I run. The goal was simple: the vault should live on hardware I own, never be reachable from the open internet, and still work from my laptop and phone wherever I am.

[Passbolt](https://www.passbolt.com/) fits that well:

- **End-to-end encrypted with OpenPGP.** Every secret is encrypted in the browser for each user who can read it. The server stores ciphertext and can't decrypt anything.
- **Built for sharing.** Folders, groups and per-item permissions, which also makes it a good store for machine credentials.
- **Has a CLI and an API**, so servers and scripts can read secrets without a human in the loop.
- **The Community Edition is free** and self-hostable.

## Table of contents

- [The design](#the-design)
- [What you need](#what-you-need)
- [Keep Docker away from the host firewall](#keep-docker-away-from-the-host-firewall)
- [Directory layout and secrets](#directory-layout-and-secrets)
- [Bind nginx to localhost](#bind-nginx-to-localhost)
- [The compose file](#the-compose-file)
- [HTTPS and remote access with Tailscale](#https-and-remote-access-with-tailscale)
- [Create the first admin](#create-the-first-admin)
- [Email](#email)
- [Backups](#backups)
- [Machine users](#machine-users)
- [Caveats](#caveats)

---

## The design

```text
Laptop / phone (Passbolt browser extension or app)
        │  Tailscale (WireGuard), tailnet only
        ▼
tailscale serve  https://<server>.<tailnet>.ts.net:8443   (real Let's Encrypt cert)
        │
        ▼  127.0.0.1:4433
passbolt container (nginx + PHP)  ──>  mariadb container 127.0.0.1:3306
        │
        └── /srv/passbolt  (database, server GPG key, JWT keys, secrets)
```

The choices that matter:

- **Nothing listens on the LAN.** Both containers use host networking but bind to `127.0.0.1` only. The single way in is Tailscale.
- **No ports opened on the router.** Tailscale works behind NAT, so the vault is invisible from the internet.
- **Docker doesn't manage the firewall.** The host's own firewall stays the only source of truth.
- **Docker instead of the Debian package.** Passbolt's package install expects a vanilla server it can configure from scratch. My box already runs other services, so containers keep Passbolt self-contained.

---

## What you need

- A small always-on Debian or Ubuntu server. A Raspberry Pi with 4 GB of RAM is enough; the Passbolt images are built for arm64 as well as amd64.
- A [Tailscale](https://tailscale.com/) account with your laptop and phone on the tailnet, and **HTTPS Certificates** enabled on the admin console's DNS page.
- An SMTP account Passbolt can send from (invites and account recovery go by email).

---

## Keep Docker away from the host firewall

By default Docker writes its own iptables rules, and those rules bypass ufw. A port you thought was firewalled can end up open to the LAN. Since every container here uses host networking, Docker doesn't need to touch the firewall at all.

```bash
sudo apt install -y docker.io docker-compose zip   # zip is for the backup script
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
sudo iptables -S | grep -i docker   # should print nothing
ip link show docker0                # should say the device doesn't exist
sudo ufw status verbose             # unchanged
```

The trade-off: default bridge networking no longer works on this host. Every container you run here needs `network_mode: host`, so treat that as a rule for the whole box. The `log-opts` cap container logs at 30 MB, which matters on an SD card.

---

## Directory layout and secrets

Everything lives under `/srv/passbolt`, which makes backups and moves easy.

```bash
sudo mkdir -p /srv/passbolt/{db,gpg,jwt,nginx,secrets}
cd /srv/passbolt

# Passwords as files, never inline in compose.yaml:
sudo sh -c 'openssl rand -base64 32 | tr -d "\n" > secrets/db_password'
sudo sh -c 'printf "%s" "you@example.com" > secrets/smtp_user'
sudo sh -c 'printf "%s" "<smtp-password>"  > secrets/smtp_password'

sudo chmod 700 secrets
sudo chmod 444 secrets/*        # readable inside the containers, the folder itself stays root-only

# Ownership for the container users:
sudo chown 2999:2999 db                           # MariaDB, see the note below
sudo chown 33:33 gpg                              # www-data inside the Passbolt image
sudo chown root:33 jwt && sudo chmod 750 jwt      # the healthcheck insists on these modes
```

**Why uid 2999 for MariaDB?** The MariaDB image runs as uid 999 by default. With host networking and bind mounts, a uid that already belongs to a host user means that user owns your database files. Check `getent passwd 999` first; I had a clash and moved MariaDB to an unused uid.

---

## Bind nginx to localhost

The non-root Passbolt image runs nginx on `8080` (HTTP) and `4433` (HTTPS) on **all** interfaces. With host networking, that means the whole LAN. Copy the two config files out of the image, bind them to `127.0.0.1`, and mount them back in.

```bash
cd /srv/passbolt
sudo docker create --name pb-tmp passbolt/passbolt:latest-ce-non-root
sudo docker cp pb-tmp:/etc/nginx/sites-available/nginx-passbolt.conf nginx/
sudo docker cp pb-tmp:/etc/nginx/snippets/passbolt-ssl.conf nginx/
sudo docker rm pb-tmp

# Drop the IPv6 listeners, bind the IPv4 ones to loopback:
sudo sed -i -E '/listen \[::\]:/d; s/listen 8080/listen 127.0.0.1:8080/' nginx/nginx-passbolt.conf
sudo sed -i -E '/listen \[::\]:/d; s/listen 4433/listen 127.0.0.1:4433/' nginx/passbolt-ssl.conf
grep -h listen nginx/*.conf     # every line should say 127.0.0.1
```

PHP-FPM inside the image listens on `127.0.0.1:9000`, which with host networking is the host's port 9000. Make sure nothing else on the host uses it.

---

## The compose file

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
      # Lets Passbolt's self-check reach its own public URL, see below:
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
sudo docker compose logs -f passbolt   # first start generates the server GPG key and JWT keys
```

**Why `extra_hosts`?** Passbolt's healthcheck calls its own `APP_FULL_BASE_URL`. If the server's DNS resolver doesn't know Tailscale's MagicDNS names, that call fails and the healthcheck complains. Mapping the name to the server's Tailscale IP inside the container fixes it without touching host DNS.

---

## HTTPS and remote access with Tailscale

`tailscale serve` terminates TLS with a real certificate for the server's tailnet name and proxies to the container on loopback. Only devices on your tailnet can reach it.

```bash
sudo tailscale serve --bg --https=8443 https+insecure://127.0.0.1:4433
sudo tailscale serve status
```

`https+insecure` is fine here. The hop it covers never leaves the machine, and the container's own certificate is self-signed. Proxying to the HTTPS port instead of `8080` avoids a redirect loop, because `PASSBOLT_SSL_FORCE` sends plain HTTP to HTTPS.

To take the vault offline instantly: `sudo tailscale serve --https=8443 off`.

---

## Create the first admin

**Gotcha:** the image's entrypoint turns the `*_FILE` variables into real environment variables only for its own process. A shell from `docker compose exec` doesn't get them, so Passbolt's CLI can't log in to the database. Export them first:

```bash
pb() {
  sudo docker compose -f /srv/passbolt/compose.yaml exec passbolt bash -c '
    export DATASOURCES_DEFAULT_PASSWORD="$(cat "$DATASOURCES_DEFAULT_PASSWORD_FILE")"
    export EMAIL_TRANSPORT_DEFAULT_USERNAME="$(cat "$EMAIL_TRANSPORT_DEFAULT_USERNAME_FILE")"
    export EMAIL_TRANSPORT_DEFAULT_PASSWORD="$(cat "$EMAIL_TRANSPORT_DEFAULT_PASSWORD_FILE")"
    /usr/share/php/passbolt/bin/cake passbolt "$@"' _ "$@"
}

pb healthcheck            # aim for "[PASS] No error found"
pb register_user -u you@example.com -f First -l Last -r admin
```

`register_user` prints a setup link. Open it on a tailnet device with the Passbolt browser extension installed, generate your key, and **store the recovery kit offline**. Without it, a lost browser profile means a lost vault.

Then install the mobile app and pair it from the extension.

---

## Email

Passbolt sends invites, recovery emails and share notifications. Most hosted SMTP providers just work with the settings in the compose file.

If you relay through a local bridge with a self-signed certificate, pin that certificate instead of turning verification off entirely:

```yaml
      PASSBOLT_PLUGINS_SMTP_SETTINGS_SECURITY_SSL_CAFILE: /etc/ssl/smtp-relay.pem
      PASSBOLT_PLUGINS_SMTP_SETTINGS_SECURITY_SSL_VERIFY_PEER_NAME: "false"
```

and mount the certificate read-only at that path. Send a test from **Administration > Email server**.

---

## Backups

A Passbolt backup has to include more than the database. The **server GPG key** and the **JWT keys** matter just as much: without the server key, a restored database is useless.

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

Then get the zips **off the box**. I pull them nightly to another machine on an encrypted disk, and a check alerts me if the newest backup is more than two days old. A backup that only exists on the server it protects is not a backup.

Test a restore at least once: fresh directory, unzip, import `passbolt.sql`, start the containers, log in.

---

## Machine users

I also use Passbolt as the single source of truth for API keys my servers and scripts need. Each machine gets its own Passbolt user and reads secrets with [go-passbolt-cli](https://github.com/passbolt/go-passbolt-cli).

One thing that surprised me: **secrets a machine user creates are invisible to everyone else** until they are shared item by item. That's end-to-end encryption doing its job: the server can't re-encrypt anything on your behalf. Plan it the other way round. Create items as yourself, in a folder, and give the machine user read-only access to that folder.

---

## Caveats

- **Your passwords are only reachable while the server and Tailscale are up.** Keep a printed or offline copy of the handful of credentials you'd need to fix the server itself.
- **Pin image versions** once it works, and read the release notes before you bump them. `latest-ce-non-root` is convenient for a first install, not for unattended upgrades.
- **Protect the recovery kit and the server GPG key** as much as the vault itself. They are the keys to everything.

Since the switch, I've cancelled the cloud subscription, and the secrets my tooling uses are synced from one place I control.
