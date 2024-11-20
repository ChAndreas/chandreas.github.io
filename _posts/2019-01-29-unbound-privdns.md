---
layout: post
title: "Private DNS Server for Android"
date:	Tue Jan 29 17:30:54 EEST 2019
---

How to create your own Private DNS Server for Android running on Docker.

**Download and install Docker**
```bash
sudo apt-get update
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  	$(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker-compose

```

**Create Let's Encrypt Certificate**
```bash
sudo apt-get update
sudo apt-get -y install certbot
sudo certbot certonly --no-eff-emai --agree-tos --email <your email> --standalone --non-interactive --domain <your domain>
cp /etc/letsencrypt/live/<your domain>/privkey.pem .
cp /etc/letsencrypt/live/<your domain>/fullchain.pem .
```
**Unbound Configuration**

```bash
vim unbound.conf
```

Inside, paste the following script:

```bash
server:
        username: "unbound"
        chroot: ""
        directory: /etc/unbound/
        pidfile: "/var/run/unbound.pid"
        logfile: "/var/log/unbound.log"
        log-local-actions: yes
        log-queries: yes
        log-replies: yes
        log-servfail: yes
        verbosity: 1
        log-queries: yes
        num-threads: 4
        so-rcvbuf: 2m
        verbosity: 2
        interface: 0.0.0.0@443
        do-ip4: yes
        do-udp: yes
        do-tcp: yes
        do-ip6: yes
        access-control: 0.0.0.0/0 allow
        tls-service-key: "/etc/unbound/privkey.pem"
        tls-service-pem: "/etc/unbound/fullchain.pem"
        tls-port: 853
        minimal-responses: yes
        cache-min-ttl: 0
        cache-max-ttl: 86400
        hide-identity: yes
        hide-version: yes
        hide-trustanchor: yes
        minimal-responses: yes
        prefetch: yes
        prefetch-key: yes
        qname-minimisation: yes
        incoming-num-tcp: 4096
        ratelimit: 1000
        num-queries-per-thread: 4096
        rrset-roundrobin: yes
        use-caps-for-id: yes
        aggressive-nsec: yes
        edns-buffer-size: 1472
        harden-glue: yes
        harden-short-bufsize: yes
        harden-large-queries: yes
        harden-algo-downgrade: yes
        harden-dnssec-stripped: yes
        harden-below-nxdomain: yes
        harden-referral-path: yes
        do-not-query-localhost: no
        statistics-cumulative: yes
        extended-statistics: yes
        val-clean-additional: yes
        ip-dscp: 18
        private-address: 10.0.0.0/8
        private-address: 172.16.0.0/12
        private-address: 192.168.0.0/16
        private-address: fd00::/8
        private-address: 169.254.0.0/16
        private-address: fe80::/10
        private-address: 127.0.0.0/8
        private-address: ::1/128
        private-address: ::ffff:0:0/96
								
forward-zone:
        name: "."
        forward-tls-upstream: yes
        forward-no-cache: no
        forward-addr: 8.8.8.8@853
        forward-addr: 8.8.4.4@853
        forward-addr: 1.1.1.1@853
        forward-addr: 1.0.0.1@853
        forward-addr: 9.9.9.9@853
remote-control:
        control-enable: no
```

**CMD Script**

```bash
vim unbound.sh
```

```bash
#!/bin/bash
/usr/sbin/unbound -d -c /etc/unbound/unbound.conf
```

**Dockerfile**

```bash
vim Dockerfile
```

Inside, paste the following script:

```bash
FROM ubuntu:24.04

ENV DEBIAN_FRONTEND noninteractive
ENV UNBOUND_VERSION 1.22.0
ENV UNBOUND_SHA256 c5dd1bdef5d5685b2cedb749158dd152c52d44f65529a34ac15cd88d4b1b3d43

SHELL ["/bin/bash", "-o", "pipefail", "-c"]

RUN set -x \
	&& apt update \
	&& apt -y upgrade \
	&& apt -y install build-essential net-tools libnghttp2-dev libssl-dev libexpat-dev libsodium-dev libevent-dev openssl wget knot-dnsutils \
	&& groupadd -g 88 unbound \
	&& useradd -c "Unbound DNS resolver" -d /var/lib/unbound -u 88 -g unbound -s /bin/false unbound \
	&& wget http://www.unbound.net/downloads/unbound-${UNBOUND_VERSION}.tar.gz \
	&& echo "${UNBOUND_SHA256}  unbound-${UNBOUND_VERSION}".tar.gz | sha256sum --check \
	&& tar -xvf unbound* \
	&& cd unbound-1.*/ \
	&& ./configure --with-libnghttp2 --with-libevent --enable-dnscrypt --prefix=/usr --sysconfdir=/etc --disable-static --with-pidfile=/run/unbound.pid \
	&& make \
	&& make install \
	&& mv -v /usr/sbin/unbound-host /usr/bin/ \
	&& unbound-anchor /etc/unbound/root.key  ; true\
	&& unbound-control-setup \
	&& unbound-checkconf \
	&& wget https://www.internic.net/domain/named.root -qO- | tee /etc/unbound/root.hints \
	&& rm -rf /unbound-${UNBOUND_VERSION}.tar.gz \
	&& rm -rf unbound-* \
	&& rm -rf /var/lib/apt/lists/* \
	&& apt-get clean

COPY ./unbound.conf /etc/unbound/unbound.conf
COPY ./privkey.pem /etc/unbound/privkey.pem
COPY ./fullchain.pem /etc/unbound/fullchain.pem
COPY ./unbound.sh /unbound.sh

WORKDIR /etc/unbound/

EXPOSE 853,443
HEALTHCHECK CMD netstat -an | grep 853 > /dev/null; if [ 0 != $? ]; then exit 1; fi;

CMD ["/unbound.sh"]
```	
**Build Image**
```bash
docker build -t unbound-tls:1.22.0 .
```
**Docker Compose**

```bash
vim docker-compose.yml
```

Inside, paste the following script:

```yaml
services:
  unbound-tls:
    image: unbound-tls:1.22.0
    container_name: unbound-tls
    restart: always
    ports:
      - "853:853"
      - "443:443"
```

**Run image**
```bash
	docker compose up -d
```