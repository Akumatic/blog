---
title: HTTPS für lokale Docker-Dienste mit Caddy und dnsmasq
date: 2025-04-23
description: |
    Der Zugriff auf lokale Docker-Dienste über localhost:port ist oft umständlich, insbesondere bei mehr als einem oder zwei Diensten.
    Der bevorzugte Weg ist die Verwendung von Subdomains und HTTPS mit einem vertrauenswürdigen Zertifikat, was mit Caddy, dnsmasq und Docker-Labels leicht zu realisieren ist.
categories:
    - services
    - network
tags:
    - docker
    - caddy
    - dnsmasq
    - reverse proxy
    - domain
    - DNS
    - HTTPS
---

Bei der lokalen Verwendung von Docker-Containern ist der übliche Weg, einen Port freizugeben und über `localhost:port` auf den Dienst zuzugreifen. Das funktioniert normalerweise, ist aber natürlich keine gute Lösung. Ein paar Nachteile sind mitunter:
- Schneller Verlust des Überblicks über zugewiesene Ports, insbesondere bei mehreren Containern
- Zugriff über HTTPS nur, wenn der Container entsprechend konfiguriert wurde - und selbst dann gibt es nur ein selbstsigniertes Zertifikat, das eine Sicherheitswarnung im Webbrowser auslöst
- Einträge von Passwortmanagern sind schwieriger zu trennen, wenn alle Dienste über `localhost` statt über eine eindeutige Domain laufen

Ein besserer Weg wäre die Verwendung von Subdomains und gültigen TLS-Zertifikaten. Es würde jedoch einen höheren Verwaltungsaufwand bedeuten, jedem Dienst ein eigenes Zertifikat von einer lokalen Zertifizierungsstelle zuzuweisen, und man müsste sich selbst um die CA und die Signierung kümmern.

Um diesen Prozess zu vereinfachen, kann ein Reverse Proxy wie **Caddy** verwendet werden.

## Reverse Proxy

Ein Reverse Proxy ist ein Server, der vor anderen Webservern sitzt und Anfragen von Clients an diese Webserver weiterleitet. Daher kümmern sie sich oft auch um sicherheitsrelevante Komponenten der Kommunikation, wie die TLS-Terminierung von HTTPS-Verbindungen.

Der Anwendungsfall in diesem Zusammenhang ist ähnlich. Wir verwenden einen Reverse-Proxy, um Anfragen an `localhost` zu verarbeiten. Dienste in Docker-Containern müssen dabei keinen Port mehr freigeben, die Kommunikation zwischen Proxy und Dienst läuft über ein internes Docker-Netzwerk.

### Warum Caddy?

[Caddy](https://caddyserver.com) ist ein moderner Webserver, der eine vereinfachte Konfiguration mit automatischem HTTPS bietet. Ein Reverse Proxy-Block in einem Caddyfile sieht beispielsweise so aus:

``` caddyfile
example.com {
    reverse_proxy localhost:3000
}
```

Mehr braucht es nicht. Caddy generiert automatisch die TLS-Zertifikate über Let's Encrypt und setzt die üblichen Header eines Reverse Proxys, was es zu einer einfachen, aber mächtigen Alternative zu den bekannten Lösungen wie Nginx macht.

### Caddy-Modul: Caddy-Docker-Proxy

Ein nützliches Modul zur Verwendung mit Docker-Diensten ist [Caddy-Docker-Proxy](https://github.com/lucaslorentz/caddy-docker-proxy). Es scannt Metadaten und sucht nach Labels, die darauf hinweisen, dass der Dienst von Caddy bedient werden soll. Aus diesen Labels wird ein Caddyfile mit den entsprechenden Einträgen erstellt, was die manuelle Verwaltung für Docker-Container überflüssig macht. Einträge für Dienste außerhalb von Docker können weiterhin über ein Caddyfile verwaltet werden.

Anweisungen, wie die Anweisungen eines Caddyfiles in Label umgewandelt werden können, findet man im [Repository](https://github.com/lucaslorentz/caddy-docker-proxy?tab=readme-ov-file#labels-to-caddyfile-conversion).

#### Beispiel

Dieses Beispiel startet *traefik/whoami* und fügt es zu einem bestehenden Proxy-Netzwerk hinzu. Nach dem Start ist der Container über `https://whoami.dev.internal` erreichbar, gesichert mit einem von Caddys interner Root CA signierten TLS-Zertifikat.

``` yml
name: whoami

services:
  whoami:
    image: traefik/whoami
    networks:
      - caddy
    labels:
      caddy: whoami.dev.internal
      caddy.tls: internal
      caddy.reverse_proxy: "{{upstream}}"

networks:
  caddy:
    external: true
```

### Caddy mit Docker Compose bereitstellen

Zuerst erstellen wir ein Proxy-Netzwerk. Dieses Netzwerk wird extern erstellt, um sicherzustellen, dass der Dienst diesem beitreten kann, auch wenn der Caddy-Stack nicht läuft.

{{< notice info >}}
Mit einem geteilten Proxy-Netzwerk können die Dienste direkt miteinander kommunizieren. Wenn dieses Verhalten verhindert werden soll, sollte für jeden Dienst ein eigenes Proxy-Netzwerk erstellt werden.
{{< /notice >}}

```
docker network create --internal caddy
```

Anschließend wird folgenden Stack in einer Datei gespeichert, der Standardname ist `docker-compose.yml`. Wenn ein anderer Name für die Datei verwendet wird, muss dieser beim Aufruf von `docker compose` explizit angegeben werden.

``` yml
name: caddy

services:
  caddy:
    container_name: caddy
    image: lucaslorentz/caddy-docker-proxy:ci-alpine
    ports:
      - 80:80
      - 443:443
    environment:
      - CADDY_INGRESS_NETWORKS=caddy
    networks:
      internal:
      proxy:
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - data:/data
    restart: unless-stopped

networks:
  internal:
    name: caddy_internal
  proxy:
    name: caddy
    external: true

volumes:
  data: 
    name: caddy_data
```

Schließlich kann der Stack mit `docker compose up -d` gestartet werden.

### Caddys Root-Zertifikat vertrauen

Damit der Computer dem von Caddy ausgestellten Zertifikat vertrauen kann, muss er der gesamten Zertifikatskette vertrauen. Mit einem lokalen Caddy könnte man `caddy trust` ausführen, um das Root-Zertifikat in den Vertrauensspeicher des Systems zu installieren.

Mit Docker ist der Container vom System isoliert und hat keinen direkten Zugriff darauf. Das Root-Zertifikat muss manuell kopiert werden und es muss in den Vertrauensspeicher des Systems oder des Browsers installiert werden. Anleitungen für Linux, Mac und Windows können in der [Dokumentation von Caddy](https://caddyserver.com/docs/running#local-https-with-docker) gefunden werden.

Für die meisten Linux-Systeme lauten die Befehle beispielsweise:

``` bash
sudo docker compose cp \
  caddy:/data/caddy/pki/authorities/local/root.crt \
  /usr/local/share/ca-certificates/caddy_docker_root.crt
sudo update-ca-certificates
```

#### Arch Linux

Die Art, wie lokale vertrauenswürdige Zertifikate gehandhabt werden, [hat sich 2014 geändert](
https://archlinux.org/news/ca-certificates-update/). Die entsprechenden Befehle für Arch Linux wären beispielsweise:

``` bash
sudo docker compose cp \
  caddy:/data/caddy/pki/authorities/local/root.crt \
  /etc/ca-certificates/trust-source/anchors/caddy_docker_root.crt
sudo trust extract-compat
```

### Ein Docker-Volume mit Caddys Root-Zertifikat erstellen

Wenn ein Container mit anderen Diensten über Caddy kommunizieren muss und dabei die Gültigkeit des Zertifikats überprüft, muss auch er der Zertifikatskette vertrauen. 

Die folgenden Befehle erstellen ein Docker-Volume namens `caddy_root_ca`, das nur das Root-Zertifikat enthält und in andere Container gemountet werden kann. Dort muss dann nur der Vertrauensspeicher aktualisiert werden, was entweder manuell oder durch Überschreiben von `entrypoint` oder `command` getan werden kann.

``` bash
docker volume create caddy_root_ca
docker compose run --rm -v $vol:/ca \
  --entrypoint "cp /data/caddy/pki/authorities/local/root.crt /ca/caddy_root.crt" \
  caddy
```

Damit der Container über Caddy auf den anderen Dienst zugreifen kann, muss ein Alias gesetzt werden.

Verkürztes, unvollständiges Beispiel für einen Dienst, der über `https://service.dev.internal` erreichbar ist und von einem anderen Container über Caddy angesprochen werden kann:

``` yml
services:
  caddy:
    image: lucaslorentz/caddy-docker-proxy:ci-alpine
    # ...
    networks:
      caddy:
      proxy:
        aliases:
          - service.dev.internal
    # ...

  web:
    # ...
    networks:
      - proxy
    labels:
      caddy: service.dev.internal
      caddy.tls: internal
      caddy.reverse_proxy: "{{upstream}}"
    # ...

  client:
    # ...
    volumes:
      - caddy_ca:/usr/local/share/ca-certificates/caddy
    networks:
      - proxy
    # ...

# ...

volumes:
  caddy_ca:
    external: true
    name: caddy_root_ca
```

Nach der Aktualisierung des Vertrauensspeichers kann 'client' nun mit 'web' über eine vertraute HTTPS-Verbindung über Caddy kommunizieren.

## Lokale Domain

Mit unserem Reverse Proxy kann für jeden Dienst eine eigene Subdomain verwendet werden.

Statt in `/etc/hosts` jeden Eintrag manuell hinterlegen zu müssen, werden wir einen Wildcard-DNS-Eintrag für die Subdomain `dev.internal` verwenden. Da `.internal` von der ICANN als Domain für private Anwendungen reserviert wurde, ist die Nutzung problemlos möglich. Durch die Reservierung ist garantiert dass die Domain nicht als Top-Level-Domain im DNS des Internets installiert wird.

Durch diesen Eintrags werden die Domain selbst und alle Subdomains nach `localhost` aufgelöst.

{{< notice info >}}
Generell kann jede beliebige Domain verwendet werden. Die Verwendung einer bestehenden, global gerouteten Domain kann zu Problemen bei der Namensauflösung und damit der Erreichbarkeit führen. 

Aber auch eine TLD, die noch nicht im DNS des Internets registriert ist, sollte nur mit Vorsicht verwendet werden, solange sie nicht wie `.internal` explizit reserviert wurde.
{{< /notice >}}

### Linux mit NetworkManager

- dnsmasq installieren
- DNS-Resolver von NetworkManager änden
``` bash
sudo bash -c 'echo "[main]" > /etc/NetworkManager/conf.d/dns.conf'
sudo bash -c 'echo "dns=dnsmasq" >> /etc/NetworkManager/conf.d/dns.conf'
```
- DNS-Einträge hinzufügen
``` bash
sudo bash -c 'echo "address=/dev.internal/127.0.0.1" > /etc/NetworkManager/dnsmasq.d/dev.internal.conf'
sudo bash -c 'echo "address=/dev.internal/::1" >> /etc/NetworkManager/dnsmasq.d/dev.internal.conf'
```
- NetworkManager neu laden
``` bash
nmcli general reload
```

### MacOS
- Wenn noch nicht geschehen: [Homebrew](https://brew.sh/) installieren
- dnsmasq installieren
``` bash
brew install dnsmasq
```
- DNS-Einträge hinzufügen
``` bash
echo "address=/dev.internal/127.0.0.1" > $(brew --prefix)/etc/dnsmasq.d/dev.internal.conf
echo "address=/dev.internal/::1" >> $(brew --prefix)/etc/dnsmasq.d/dev.internal.conf
```
- Autostart aktivieren
``` bash
sudo brew services start dnsmasq
```
- Zu Resolvern hinzufügen
```
sudo mkdir -v /etc/resolver
sudo bash -c 'echo "nameserver 127.0.0.1" > /etc/resolver/internal'
```

## Setup testen

Zunächst sollte getestet werden, ob die Domain-Auflösung wie vorgesehen funktioniert. Dazu ein Terminal öffnen und `dig <domain>` (Linux) oder `dscacheutil -q host -a name <domain>` (MacOS) verwenden.

Sowohl für die Domain selbst (z.B. `dev.internal`) als auch für eine beliebige Subdomain davon (z.B. `a.dev.internal`) sollten `127.0.0.1` für IPv4 und `::1` für IPv6 zurückkommen.

Dann kann das oben erwähnten Beispiel [„whoami“](#beispiel) gestartet werden. Nach dem Start des Containers sollte der Dienst über die angegebene URL (wie z.B. https://whoami.dev.internal) mit gültigen HTTPS ohne Sicherheitswarnungen bedient werden.
