---
lang: pt
page_id: pi_hole_home_network
permalink: /posts/pi-hole-home-network
title: Protegendo uma Rede Doméstica com Pi-hole, Unbound e Tailscale
date: 2026-10-21
categories: [Raspberry Pi, Networking, Security, Self-Hosting]
tags: [Tutorial, Self-Hosting, Raspberry Pi, Pi-hole, Unbound, Tailscale, DNS, DHCP]
author: Angel
description: Transforme um Raspberry Pi em um bloqueador de anúncios, rastreadores e malware para a casa inteira, com seu próprio resolvedor DNS recursivo, e mantenha a proteção no celular e no notebook fora de casa com o Tailscale.
image:
  path: /assets/img/pi-hole.jpg
  alt: Uma placa Raspberry Pi 4 sobre uma superfície escura
---

## Protegendo uma Rede Doméstica com Pi-hole, Unbound e Tailscale

Todo dispositivo de uma casa pergunta a um servidor DNS para onde ir antes de carregar qualquer coisa: celulares, notebooks, a TV, o medidor inteligente, a placa ESP32 barata guardada numa gaveta. Quem responde a essas perguntas vê sua navegação e decide o que é resolvido. Por padrão, esse alguém é o seu provedor de internet.

Este guia transfere essa tarefa para um Raspberry Pi que você controla:

- O **Pi-hole** bloqueia anúncios, rastreadores, phishing, golpes e domínios de malware para todos os dispositivos da rede, sem nenhum aplicativo para instalar.
- O **Unbound** resolve os nomes por conta própria, direto dos servidores raiz, com validação DNSSEC. Nenhum resolvedor do provedor, do Google ou da Cloudflare no meio.
- O **Tailscale** leva a mesma filtragem para o seu celular e notebook quando você está na rua, sem abrir uma única porta no roteador.
- **Um pendrive USB** guarda um clone inicializável do cartão SD, então um cartão que morreu se resolve em dois minutos.

## Índice

- [Como tudo se encaixa](#como-tudo-se-encaixa)
- [Hardware](#hardware)
- [Gravar o sistema operacional](#gravar-o-sistema-operacional)
- [IP estático e somente Ethernet](#ip-estático-e-somente-ethernet)
- [Reforço básico de segurança](#reforço-básico-de-segurança)
- [Instalar o Pi-hole](#instalar-o-pi-hole)
- [Adicionar o Unbound como resolvedor upstream](#adicionar-o-unbound-como-resolvedor-upstream)
- [Listas de bloqueio](#listas-de-bloqueio)
- [Deixar o Pi-hole distribuir endereços (DHCP)](#deixar-o-pi-hole-distribuir-endereços-dhcp)
- [Fechar o vazamento de DNS por IPv6](#fechar-o-vazamento-de-dns-por-ipv6)
- [Leve com você: Tailscale](#leve-com-você-tailscale)
- [HTTPS para a página de administração](#https-para-a-página-de-administração)
- [Backups](#backups)
- [Solução de problemas](#solução-de-problemas)
- [O que eu melhoraria a seguir](#o-que-eu-melhoraria-a-seguir)

---

## Como tudo se encaixa

```text
Internet ── Roteador do provedor (gateway, DHCP DESLIGADO na LAN principal)
              ├── todos os dispositivos da casa (recebem IP e DNS do Pi)
              ├── Wi-Fi de convidados (DHCP do roteador ainda ligado, isolado, sem filtragem)
              └── Raspberry Pi 192.168.1.53 (Ethernet)
                    Pi-hole (DNS :53, DHCP) ──> Unbound 127.0.0.1:5335 ──> servidores raiz
                    Tailscale 100.x.y.z (DNS dos seus dispositivos fora de casa, exit node)
                    Pendrive USB = clone semanal inicializável do cartão SD
```

O Pi fica **ao lado** do roteador, não entre o roteador e a internet. O roteador do provedor continua sendo o gateway. Um firewall em linha (segunda placa de rede, OpenWrt) é mais poderoso, mas é exagero para um único Pi e transforma cada reinicialização do Pi em uma queda para a casa inteira.

Os endereços neste post são exemplos. Escolha para o Pi um endereço **fora** do pool DHCP do seu roteador (aqui o pool vai de `.64` a `.253`, então `.53` é seguro).

---

## Hardware

- **Raspberry Pi 4 ou 5**. 2 GB de RAM são mais que suficientes para este trabalho.
- **Fonte de alimentação oficial**. Pis com energia insuficiente corrompem cartões SD.
- **Cartão microSD**, de 16 GB ou mais.
- **Cabo Ethernet** para uma porta LAN livre no roteador.
- **Pendrive USB**, pelo menos do tamanho do cartão SD, para o clone de backup.

---

## Gravar o sistema operacional

Use o **Raspberry Pi Imager** e escolha **Raspberry Pi OS Lite (64-bit)**. Antes de gravar, abra as configurações de personalização:

- **Hostname**: escolha um e anote. Você vai usar SSH em `<hostname>.local`, não em `pihole.local`. Eu perdi dez minutos nisso.
- **User**: seu próprio nome de usuário, não `pi`.
- **SSH**: ativado, **somente chave pública**. Cole a chave de `~/.ssh/id_ed25519.pub`.
- **Raspberry Pi Connect**: desligado. Esta caixa não deve ligar para casa com nada que você não precise.

Se você está reaproveitando hardware, espere ver `REMOTE HOST IDENTIFICATION HAS CHANGED` na primeira conexão. Essa é a chave de host antiga; limpe-a com `ssh-keygen -R <ip-or-hostname>`, e só faça isso quando souber que você regravou o cartão.

---

## IP estático e somente Ethernet

O DNS da casa inteira não deve depender de Wi-Fi. Defina um endereço estático na Ethernet com o NetworkManager. **Por enquanto, aponte o DNS do próprio Pi para o roteador**: configurá-lo como `127.0.0.1` antes de o Pi-hole existir quebra o instalador do Pi-hole.

```bash
nmcli con show                       # encontre o nome do perfil Ethernet (geralmente netplan-eth0)
sudo nmcli con mod netplan-eth0 \
  ipv4.method manual \
  ipv4.addresses 192.168.1.53/24 \
  ipv4.gateway 192.168.1.1 \
  ipv4.dns 192.168.1.1
sudo nmcli con up netplan-eth0
```

Depois desligue os rádios para que o Pi tenha exatamente um caminho até a rede. Um Pi com Ethernet e Wi-Fi na mesma sub-rede escolhe uma interface por conexão, e já vi isso causar timeouts de DNS sob carga em uma versão mais antiga.

```bash
echo "dtoverlay=disable-wifi" | sudo tee -a /boot/firmware/config.txt
echo "dtoverlay=disable-bt"   | sudo tee -a /boot/firmware/config.txt
sudo reboot
```

---

## Reforço básico de segurança

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y unattended-upgrades ufw
sudo dpkg-reconfigure -plow unattended-upgrades   # patches de segurança automáticos
```

Firewall: negue tudo que entra, depois abra apenas o que a LAN precisa.

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 192.168.1.0/24 to any port 22 proto tcp     # SSH
sudo ufw allow from 192.168.1.0/24 to any port 53               # DNS
sudo ufw allow from 192.168.1.0/24 to any port 80,443 proto tcp # página de administração
sudo ufw allow from 192.168.1.0/24 to any port 5353 proto udp   # mDNS, para que <hostname>.local funcione
sudo ufw allow 67/udp                                           # DHCP, veja abaixo
sudo ufw enable
```

Duas regras são fáceis de errar:

- **O DHCP (67/udp) deve ser permitido de qualquer origem.** Um dispositivo que pede um endereço ainda não tem um, então a requisição vem de `0.0.0.0`. Uma regra `from 192.168.1.0/24` nunca a captura.
- **O mDNS (5353/udp)** é o que faz `<hostname>.local` resolver. Sem ele, você vai digitar o IP para sempre.

---

## Instalar o Pi-hole

```bash
curl -sSL https://install.pi-hole.net | bash
```

Aceite os padrões; qualquer upstream serve por enquanto, porque o Unbound o substitui no próximo passo. Defina a senha de administrador depois:

```bash
sudo pihole setpassword
```

O Pi-hole v6 guarda suas configurações em `/etc/pihole/pihole.toml`. Leia ou altere qualquer chave com `sudo pihole-FTL --config <key> [value]` em vez de editar o arquivo à mão.

Agora que o Pi-hole responde, aponte o Pi para si mesmo:

```bash
sudo nmcli con mod netplan-eth0 ipv4.dns 127.0.0.1
sudo nmcli con up netplan-eth0
```

---

## Adicionar o Unbound como resolvedor upstream

O Pi-hole só filtra. Alguém ainda precisa resolver os nomes que ele deixa passar. O Unbound faz isso localmente, perguntando aos servidores raiz, depois aos servidores `.com`, depois aos servidores do próprio domínio, e verificando as assinaturas DNSSEC pelo caminho.

```bash
sudo apt install -y unbound
```

**Pegadinha:** logo após a instalação, o Unbound inicia com a configuração padrão na porta 53, colide com o Pi-hole, falha algumas vezes, e o systemd limita a taxa de tentativas (`Start request repeated too quickly`). Um restart simples é recusado. Escreva a configuração primeiro, depois limpe o estado de falha.

Crie `/etc/unbound/unbound.conf.d/pi-hole.conf` (esta é a receita da documentação do Pi-hole, resumida):

```conf
server:
    verbosity: 0
    interface: 127.0.0.1
    port: 5335
    do-ip4: yes
    do-udp: yes
    do-tcp: yes
    do-ip6: no
    prefer-ip6: no

    harden-glue: yes
    harden-dnssec-stripped: yes
    use-caps-for-id: no
    edns-buffer-size: 1232
    prefetch: yes
    num-threads: 1
    so-rcvbuf: 1m

    private-address: 192.168.0.0/16
    private-address: 169.254.0.0/16
    private-address: 172.16.0.0/12
    private-address: 10.0.0.0/8
    private-address: fd00::/8
    private-address: fe80::/10
```

```bash
sudo systemctl disable --now unbound-resolvconf.service
sudo systemctl reset-failed unbound
sudo systemctl restart unbound

# Deve retornar um endereço:
dig example.com @127.0.0.1 -p 5335
# Verificação de DNSSEC, deve retornar SERVFAIL (a assinatura está propositalmente quebrada):
dig fail01.dnssec.works @127.0.0.1 -p 5335
```

Aponte o Pi-hole para ele e descarte todos os outros upstreams:

```bash
sudo pihole-FTL --config dns.upstreams '["127.0.0.1#5335"]'
```

---

## Listas de bloqueio

A lista StevenBlack padrão é um bom começo. Eu adicionei três da [HaGeZi](https://github.com/hagezi/dns-blocklists), que o Pi-hole v6 lê no formato estilo ABP da pasta `adblock/`:

| Lista | O que bloqueia | Tamanho aproximado |
|---|---|---|
| StevenBlack (padrão) | anúncios, rastreadores | ~70k |
| HaGeZi Pro | anúncios, rastreadores, telemetria, com poucas quebras | ~200k |
| HaGeZi Threat Intelligence Feeds | malware, phishing, golpes, comando e controle | ~2M |
| HaGeZi Fake | lojas falsas e sites de golpe | ~17k |

Adicione-as em **Lists** na página de administração:

```text
https://cdn.jsdelivr.net/gh/hagezi/dns-blocklists@latest/adblock/pro.txt
https://cdn.jsdelivr.net/gh/hagezi/dns-blocklists@latest/adblock/tif.txt
https://cdn.jsdelivr.net/gh/hagezi/dns-blocklists@latest/adblock/fake.txt
```

**Pegadinha:** uma lista adicionada pela interface mostra um ponto de interrogação cinza e **não bloqueia nada** até o gravity rodar. Uma das minhas listas ficou sem carregar por um dia antes de eu perceber. Rode-o imediatamente:

```bash
sudo pihole -g
```

Depois disso, o cron semanal do próprio Pi-hole atualiza as listas. Não edite `/etc/cron.d/pihole`; o `pihole -up` o reescreve.

Você também vai ver linhas vermelhas de `_dns.resolver.arpa` no Query Log. Isso é esperado: o Pi-hole bloqueia a consulta que Apple e Android usam para descobrir DNS criptografado, o que de outra forma permitiria que os dispositivos o contornassem silenciosamente.

---

## Deixar o Pi-hole distribuir endereços (DHCP)

A configuração usual é dizer ao servidor DHCP do roteador para entregar o Pi como servidor DNS. O roteador do meu provedor mostra esses campos de DNS, mas os **bloqueia**. Se o seu também faz isso, o caminho é mover o DHCP para o Pi.

1. Anote a faixa DHCP atual do roteador, o tempo de concessão e o gateway.
2. Configure o DHCP do Pi-hole com os mesmos valores, mas deixe-o desligado:

   ```bash
   sudo pihole-FTL --config dhcp.start 192.168.1.64
   sudo pihole-FTL --config dhcp.end 192.168.1.253
   sudo pihole-FTL --config dhcp.router 192.168.1.1
   sudo pihole-FTL --config dhcp.netmask 255.255.255.0
   sudo pihole-FTL --config dhcp.leaseTime 24h
   ```

3. **Desligue** o DHCP do roteador para a rede principal.
4. Imediatamente **ligue** o do Pi-hole:

   ```bash
   sudo pihole-FTL --config dhcp.active true
   ```

5. Desligue e ligue o Wi-Fi em um celular. Ele deve aparecer em **Settings > DHCP** na página de administração e começar a aparecer no Query Log.

Deixei a **rede de convidados** no DHCP do próprio roteador. Convidados e gadgets de IoT ficam isolados da LAN principal e simplesmente não recebem filtragem.

### A contrapartida e a saída de emergência

O Pi agora distribui todos os endereços IP da casa. Se ele morrer, os dispositivos não conseguem entrar na rede. Anote o procedimento de reversão em um lugar que uma pessoa não técnica da casa consiga encontrar:

1. Abra a página de administração do roteador, **ligue o DHCP novamente** para a rede principal e salve.
2. Desligue e ligue o Wi-Fi em cada dispositivo para que ele pegue uma concessão do roteador.
3. Se o Pi ainda estiver vivo, pare o DHCP dele para que dois servidores não briguem: `sudo pihole-FTL --config dhcp.active false`.

Isso leva cerca de um minuto e devolve a você uma internet comum, sem filtragem. Aproveite e cole no Pi uma etiqueta "Não desligue: internet de casa".

---

## Fechar o vazamento de DNS por IPv6

Depois de tudo isso, ainda apareciam anúncios no meu notebook de vez em quando. O motivo: o roteador do provedor anuncia IPv6 e se oferece **ele mesmo** como servidor DNS IPv6, e não existe configuração para desligar isso. Os dispositivos o usam sem reclamar e pulam o Pi-hole.

Verifique em um Mac:

```bash
scutil --dns | grep nameserver
```

Se um endereço IPv6 que você não reconhece estiver perto do topo, você está vazando. Opções, da melhor para a pior:

- **Rode o Tailscale no dispositivo** com a substituição de DNS descrita abaixo. Ele vira o resolvedor número 1 e encaminha para o Pi. Isso resolveu nos meus próprios dispositivos.
- Desligue o IPv6 por dispositivo: `networksetup -setv6off Wi-Fi` (desfaça com `networksetup -setv6automatic Wi-Fi`).
- Coloque seu próprio roteador atrás do do provedor. A solução de verdade, e a que dá mais trabalho.

Verifique também os outros desvios comuns: o "Secure DNS" do navegador (DNS sobre HTTPS) no Chrome e no Firefox, e o iCloud Private Relay.

---

## Leve com você: Tailscale

Escolhi o Tailscale em vez de um WireGuard ou PiVPN auto-hospedado porque ele **não precisa de redirecionamento de portas**, funciona atrás de CGNAT e meus dispositivos já estavam nele.

```bash
curl -fsSL https://tailscale.com/install.sh | sh

# Permita que o Pi encaminhe tráfego, necessário para o exit node:
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/99-tailscale.conf
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf

# Não deixe o Tailscale reescrever o DNS do próprio Pi; anuncie-o como exit node:
sudo tailscale up --accept-dns=false --advertise-exit-node
```

O Pi-hole precisa responder na interface do Tailscale, e o ufw precisa deixar o tráfego entrar:

```bash
sudo pihole-FTL --config dns.listeningMode ALL   # o ufw mantém a internet de fora
sudo ufw allow in on tailscale0
sudo ufw route allow in on tailscale0 out on eth0  # tráfego do exit node
```

No console de administração do Tailscale:

- **Machines > (your Pi) > Edit route settings**: aprove **Use as exit node** e desative a expiração de chave para esta máquina.
- **DNS**: adicione o IP Tailscale do Pi como nameserver global e ligue **Override DNS servers**.

Agora todo dispositivo com Tailscale resolve pelo Pi, em casa ou no Wi-Fi de um hotel. Teste a partir de um notebook: `dig doubleclick.net @100.x.y.z` deve retornar `0.0.0.0`.

### Duas pegadinhas de política

Se a sua tailnet tem regras de acesso personalizadas (a minha tinha, por causa de um servidor com tag), os padrões deixam de valer:

- **Dispositivos com tag perdem o DNS.** Uma substituição de DNS em toda a tailnet vale para todo dispositivo que aceita DNS, inclusive servidores com tag. Se a política não os deixa alcançar o Pi na porta 53, toda consulta nessas máquinas dá timeout. Adicione um grant:

  ```json
  {"src": ["tag:server"], "dst": ["<pi-tailscale-ip>"], "ip": ["udp:53", "tcp:53"]}
  ```

- **O exit node não aparece nos clientes**, embora o console mostre o selo. Os membros precisam de permissão para usar a internet através dele:

  ```json
  {"src": ["autogroup:member"], "dst": ["autogroup:internet"], "ip": ["*"]}
  ```

No celular, o controle que você quer é **Exit Node > (your Pi)**. "Run as Exit Node" transforma o próprio celular em um exit node; ignore.

---

## HTTPS para a página de administração

De fábrica, a página de administração usa um certificado autoassinado, e a página de backup avisa que as exportações trafegam em texto puro. O Tailscale pode emitir um certificado Let's Encrypt de verdade para o nome do Pi na tailnet. Ative **HTTPS Certificates** na página DNS do console, depois:

```bash
sudo tailscale cert \
  --cert-file /etc/pihole/ts.crt \
  --key-file /etc/pihole/ts.key \
  <pi-name>.<your-tailnet>.ts.net
sudo sh -c 'cat /etc/pihole/ts.key /etc/pihole/ts.crt > /etc/pihole/tls-tailscale.pem'
sudo chown pihole:pihole /etc/pihole/tls-tailscale.pem
sudo chmod 600 /etc/pihole/tls-tailscale.pem

sudo pihole-FTL --config webserver.tls.cert /etc/pihole/tls-tailscale.pem
sudo pihole-FTL --config webserver.domain <pi-name>.<your-tailnet>.ts.net
sudo pihole-FTL --config webserver.port '80r,443s'   # r = redireciona para HTTPS, s = TLS
sudo systemctl restart pihole-FTL
```

O certificado dura 90 dias. Eu embrulhei esses comandos em um pequeno script que só reinicia o Pi-hole quando o certificado realmente mudou, e o executo mensalmente a partir de `/etc/cron.d`.

---

## Backups

Duas camadas.

**1. Um clone inicializável do cartão SD inteiro** com o [rpi-clone](https://github.com/geerlingguy/rpi-clone). Se o cartão morrer, tire-o e ligue: o Pi 5 tenta o cartão SD primeiro, depois o USB, então ele inicializa direto do pendrive.

```bash
curl https://raw.githubusercontent.com/geerlingguy/rpi-clone/master/install | sudo bash
sudo systemd-run --unit=rpi-clone /usr/local/sbin/rpi-clone -u sda
journalctl -u rpi-clone   # procure por "Done with clone to /dev/sda"
```

Rode os clones manuais pelo `systemd-run`. Se você o rodar numa sessão SSH comum e a conexão cair, você nunca descobre se ele terminou. Para o cron semanal, envie a saída para o journal para que as falhas fiquem visíveis:

```bash
# /etc/cron.d/rpi-clone
0 4 * * 0 root /usr/bin/systemd-cat -t rpi-clone /usr/local/sbin/rpi-clone -u sda
```

**2. A exportação do próprio Pi-hole (Teleporter)**: configurações, listas, entradas de permissão/negação e concessões DHCP em um único zip. Crie um por semana e copie-o **para fora do Pi**:

```bash
# /etc/cron.d/pihole-teleporter
30 3 * * 0 root cd /var/backups/pihole && /usr/bin/pihole-FTL --teleporter && find /var/backups/pihole -name '*.zip' -mtime +28 -delete
```

Restaure em **Settings > Teleporter > Import**, ou com `sudo pihole-FTL --teleporter <file.zip>`. Trate o zip como sensível: ele contém o hash da senha de administrador.

---

## Solução de problemas

| Sintoma | Primeira coisa a verificar |
|---|---|
| A casa inteira sem internet | Faça a [reversão](#a-contrapartida-e-a-saída-de-emergência) primeiro, depure depois. Em seguida, `ping` no Pi, `systemctl status pihole-FTL unbound`, `vcgencmd get_throttled` (`0x0` significa nenhum problema de energia ou calor). |
| O Pi está no ar mas as páginas não carregam | `dig example.com @127.0.0.1 -p 5335` no Pi. Se falhar, o Unbound está fora do ar: `sudo systemctl restart unbound`. |
| Um site está quebrado | Query Log, filtre pelo IP do dispositivo, encontre a linha vermelha, clique em **Allow**. Ou `pihole allow <domain>`. `pihole disable 5m` para um teste rápido. |
| Um dispositivo não recebe endereço IP | `sudo pihole-FTL --config dhcp.active` é `true`, e `sudo ufw status` mostra `67/udp ALLOW Anywhere`. |
| Anúncios em um dispositivo em casa | O vazamento de IPv6 acima, ou DNS sobre HTTPS no navegador. |
| Sem bloqueio fora de casa | Tailscale conectado? Override DNS ligado? `dns.listeningMode` é `ALL`? |

---

## O que eu melhoraria a seguir

- **Um pequeno nobreak (UPS)** para o roteador e o Pi. Uma queda de energia repentina é a forma mais provável de corromper o cartão SD.
- **Um segundo Pi-hole** para que DNS e DHCP sobrevivam à queda de uma caixa.
- **Mover os gadgets de IoT para a rede de convidados.** Um medidor inteligente e uma placa de hobby não têm por que estar na mesma LAN do meu notebook.
- **Reforçar o próprio roteador**: nova senha de administrador, WPS, UPnP e administração remota desligados, WPA3 onde os dispositivos suportarem.

Depois disso, a manutenção é quase zero: os patches de segurança se instalam sozinhos, as listas de bloqueio são atualizadas toda semana, e um `pihole -up` por mês mantém o Pi-hole atualizado.
