---
title: HTTPS for Docker services running locally with Caddy and dnsmasq
date: 2025-04-23
description: |
    Accessing local Docker services via localhost:port is often cumbersome, especially with more than one or two services.
    The preferred way is to use subdomains and HTTPS with a trusted certificate, which is easy to realize with Caddy, dnsmasq and Docker labels.
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

When using Docker containers locally, the usual way is to expose a port and access the service via `localhost:port`. This usually works, but is of course not a good solution. A few downsides are:
- Quick loss of overview of assigned ports, especially with more containers
- Access via HTTPS only if the container has been configured accordingly - and even then there is only a self-signed certificate that triggers a security warning in the web browser
- Entries of password managers are more difficult to separate if all services run via `localhost` instead of a unique domain

A better way would be to use subdomains and valid TLS certificates. However, giving each service its own signed certificate from a local certificate authority would mean more administrative work before the service can be used, and you would also have to take care of the certificate authority and signing yourself.

To simplify this process, a reverse proxy like **Caddy** can be used.

## Reverse Proxy

A reverse proxy is a server that sits in front of webservers and forwards requests from clients to these webservers. As a result, they often also take care of security-relevant components of communication such as TLS termination of HTTPS connections.

The use case in this context is similar. We use a reverse proxy to handle requests to `localhost`. Services in Docker containers do not need to expose a port anymore, the communication between proxy and service runs via an internal Docker network

### Why Caddy?

[Caddy](https://caddyserver.com) is a modern web server offering a simplified configuration with automatic HTTPS. A reverse proxy block in a Caddyfile looks like this, for example:

``` caddyfile
example.com {
    reverse_proxy localhost:3000
}
```

That's it. Caddy automatically generates the TLS certificates via Let's Encrypt and sets the usual headers of a reverse proxy, which makes it a simple but powerful alternative to the well-known solutions such as Nginx.

### Caddy Module: Caddy-Docker-Proxy

A useful module to use with docker services is [Caddy-Docker-Proxy](https://github.com/lucaslorentz/caddy-docker-proxy). It scans metadata and searches for labels that indicate that the service should be served by Caddy. A Caddyfile with the corresponding entries is created from these labels, which makes manual management for Docker containers superfluous. Entries for services outside of Docker can still be managed via Caddyfile.

Instructions how to convert the Caddyfile instructions to labels can be found in the [repository](https://github.com/lucaslorentz/caddy-docker-proxy?tab=readme-ov-file#labels-to-caddyfile-conversion).

#### Example

This example spins up traefik/whoami and adds it to an existing caddy proxy network. After creation, the container is reachable via `https://whoami.dev.internal`, secured with an TLS certificate signed by Caddy's internal Root CA.

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

### Setup Caddy with Docker Compose

First, we create a proxy network. This network is created externally to ensure that a service can join it even if the Caddy stack is not running.

{{< notice info >}}
With a shared proxy network, the services can communicate directly with each other. If you want to prevent this behavior, you should create a proxy network for each service.
{{< /notice >}}

```
docker network create --internal caddy
```

We then store the following stack in a file, the default name is `docker-compose.yml`. If a different name is used for the file, this must be explicitly specified when calling `docker compose`.

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

Finally, the stack can be started with `docker compose up -d`.

### Trust Caddy's Root CA

For your computer to trust the certificate issued by Caddy, it needs to trust the certificate chain. With a local Caddy, one could run `caddy trust` to install the root CA to the system's trust store.

With docker, the container is isolated from the system and has no direct access to it. The root certificate needs to be copied and trusted manually. Instructions for Linux, Mac and Windows can be found in the [Caddy documentation](https://caddyserver.com/docs/running#local-https-with-docker).

For most Linux the commands are:

``` bash
sudo docker compose cp \
  caddy:/data/caddy/pki/authorities/local/root.crt \
  /usr/local/share/ca-certificates/caddy_docker_root.crt
sudo update-ca-certificates
```

#### Arch Linux

The way local CA certificates are handled [has changed in 2014](
https://archlinux.org/news/ca-certificates-update/). The corresponding commands would be:

``` bash
sudo docker compose cp \
  caddy:/data/caddy/pki/authorities/local/root.crt \
  /etc/ca-certificates/trust-source/anchors/caddy_docker_root.crt
sudo trust extract-compat
```

### Create a docker volume with Caddy's Root CA

If a container needs to communicate with other services via caddy and checks the validity of the certificate, it also needs to trust the certificate chain. 

The following commands create a docker volume named `caddy_root_ca` that contains only the root CA and can be mounted in other containers. There, only the trust store needs to be updated, which can be triggered either manually or by overriding `entrypoint` or `command`.

``` bash
docker volume create caddy_root_ca
docker compose run --rm -v $vol:/ca \
  --entrypoint "cp /data/caddy/pki/authorities/local/root.crt /ca/caddy_root.crt" \
  caddy
```

For the container to access the other service via Caddy, an alias needs to be set.

Shortened incomplete example for a service that is accessible via `https://service.dev.internal` and can be accessed by another docker service via Caddy:

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

After running the trust store update, the ‘client’ service can now communicate with ‘web’ over a trusted HTTPS connection via Caddy.

## Local domain

With our reverse proxy, an individual subdomain can be used for each service.

Instead of having to enter each entry manually in `/etc/hosts`, we are going to use a wildcard DNS entry for the subdomain `dev.internal`. Since `.internal` has been reserved by ICANN as a domain name for private application use, the top level domain is safe to use, as it is guaranteed that it will not be installed in the Internet's DNS.

By specifying this entry, the domain itself and all subdomains are resolved to `localhost`.

{{< notice info >}}
Generally, any domain can be used. Using an existing, globally routed domain can cause problems with name resolution and therefore accessibility. 

But even a TLD that is not yet registered in the DNS of the Internet should only be used with caution as long as it has not been explicitly reserved like `.internal`.
{{< /notice >}}

### Linux with NetworkManager

- Install dnsmasq
- Change DNS resolver of NetworkManager
``` bash
sudo bash -c 'echo "[main]" > /etc/NetworkManager/conf.d/dns.conf'
sudo bash -c 'echo "dns=dnsmasq" >> /etc/NetworkManager/conf.d/dns.conf'
```
- Add DNS entries
``` bash
sudo bash -c 'echo "address=/dev.internal/127.0.0.1" > /etc/NetworkManager/dnsmasq.d/dev.internal.conf'
sudo bash -c 'echo "address=/dev.internal/::1" >> /etc/NetworkManager/dnsmasq.d/dev.internal.conf'
```
- Reload NetworkManager
``` bash
nmcli general reload
```

### MacOS
- If not already done: Install [Homebrew](https://brew.sh/)
- Install dnsmasq
``` bash
brew install dnsmasq
```
- Add DNS entries
``` bash
echo "address=/dev.internal/127.0.0.1" > $(brew --prefix)/etc/dnsmasq.d/dev.internal.conf
echo "address=/dev.internal/::1" >> $(brew --prefix)/etc/dnsmasq.d/dev.internal.conf
```
- Enable autostart
``` bash
sudo brew services start dnsmasq
```
- Add to resolvers
```
sudo mkdir -v /etc/resolver
sudo bash -c 'echo "nameserver 127.0.0.1" > /etc/resolver/internal'
```

## Test the setup

First check whether the domain resolution is working as intended. Open a terminal and use `dig <domain>` (Linux) or `dscacheutil -q host -a name <domain>` (MacOS). 

Us the command both for the domain itself (e.g. `dev.internal`) and for a subdomain of it (e.g. `a.dev.internal`). Both the domain itself (e.g. `dev.internal`) and for any subdomain of it (e.g. `a.dev.internal`) should return `127.0.0.1` for IPv4 and `::1` for IPv6.

You can then start the example service [“whoami”](#example) mentioned above. After starting the container, call up the specified URL (e.g. https://whoami.dev.internal). It should be served with a valid HTTPS without security warnings.
