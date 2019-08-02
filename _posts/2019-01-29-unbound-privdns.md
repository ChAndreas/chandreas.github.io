---
layout: post
title: "Private DNS Server for Android 9"
date:	Tue Jan 29 17:30:54 EEST 2019
---

Android 9 also known as Android Pie comes with many new features but most importantly full support for DNS over TLS.

How to create your own Private DNS Server for Android 9 running on Docker.

**Download and install Docker**

	sudo apt-get update
	sudo apt-get -y install \
	apt-transport-https \
	ca-certificates \
	curl \
	gnupg-agent \
	software-properties-common
	curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
	sudo add-apt-repository \
	"deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
	sudo apt-get update
	sudo apt-get -y install docker-ce docker-ce-cli containerd.io docker-compose

**Create Let's Encrypt Certificate**

	sudo apt-get -y update
	sudo apt-get -y install software-properties-common
	sudo add-apt-repository -y ppa:certbot/certbot
	sudo apt-get update
	sudo apt-get -y install certbot
	sudo certbot certonly --no-eff-emai --agree-tos --email <your email> --standalone --non-interactive --domain <your domain>
	cp /etc/letsencrypt/live/<your domain>/privkey.pem .
	cp /etc/letsencrypt/live/<your domain>/fullchain.pem .

**Unbound Configuration**

```
vim unbound.conf
```

Inside, paste the following script:

```
server:
        username: "unbound"
        chroot: /etc/unbound/
        directory: /etc/unbound/
        pidfile: "/var/run/unbound.pid"
        use-syslog: yes
        num-threads: 4
        verbosity: 2
        interface: 0.0.0.0@853
        access-control: 0.0.0.0/0 allow
        tls-service-key: "/etc/unbound/privkey.pem"
        tls-service-pem: "/etc/unbound/fullchain.pem"
        tls-port: 853
        minimal-responses: yes
        cache-max-ttl: 14400
        cache-min-ttl: 900
        do-tcp: yes
        hide-identity: yes
        hide-version: yes
        minimal-responses: yes
        prefetch: yes
        prefetch-key: yes
        qname-minimisation: yes
        incoming-num-tcp: 4096
        num-queries-per-thread: 4096
        rrset-roundrobin: yes
        use-caps-for-id: yes
        harden-glue: yes
        harden-short-bufsize: yes
        harden-large-queries: yes
        harden-algo-downgrade: yes
        harden-dnssec-stripped: yes
        harden-below-nxdomain: yes
        harden-referral-path: no
        do-not-query-localhost: no
        statistics-cumulative: yes
        extended-statistics: yes
forward-zone:
        name: "."
        forward-tls-upstream: yes
        forward-addr: 8.8.8.8@853
        forward-addr: 8.8.4.4@853
        forward-addr: 1.1.1.1@853
        forward-addr: 1.0.0.1@853
        forward-addr: 9.9.9.9@853
remote-control:
        control-enable: no
```

**CMD Script**

```
vim unbound.sh
```

```bash
#!/bin/bash
/usr/sbin/unbound -d -c /etc/unbound/unbound.conf
```

**Dockerfile**

```
vim Dockerfile
```

Inside, paste the following script:

```
FROM ubuntu:18.04
MAINTAINER Andreas Christoforou

ENV DEBIAN_FRONTEND noninteractive
ENV UNBOUND_VERSION 1.9.2
ENV UNBOUND_SHA256 6f7acec5cf451277fcda31729886ae7dd62537c4f506855603e3aa153fcb6b95

RUN set -x \
	&& apt update \
	&& apt -y upgrade \
	&& apt -y install build-essential libssl-dev libexpat-dev libsodium-dev libevent-dev openssl wget  knot-dnsutils \
	&& groupadd -g 88 unbound \
	&& useradd -c "Unbound DNS resolver" -d /var/lib/unbound -u 88 -g unbound -s /bin/false unbound \
	&& wget http://www.unbound.net/downloads/unbound-${UNBOUND_VERSION}.tar.gz \
	&& echo "${UNBOUND_SHA256}  unbound-${UNBOUND_VERSION}".tar.gz | sha256sum --check \
	&& tar -xvf unbound* \
	&& cd unbound-1.*/ \
	&& ./configure --with-libevent --enable-dnscrypt --prefix=/usr --sysconfdir=/etc --disable-static --with-pidfile=/run/unbound.pid \
	&& make \
	&& make install \
	&& mv -v /usr/sbin/unbound-host /usr/bin/ \
	&& unbound-anchor /etc/unbound/root.key  ; true\
	&& unbound-control-setup \
	&& unbound-checkconf \
	&& rm -rf /unbound-${UNBOUND_VERSION}.tar.gz \
	&& rm -rf unbound-* \
	&& rm -rf /var/lib/apt/lists/* \
	&& apt-get clean

COPY ./unbound.conf /etc/unbound/unbound.conf
COPY ./privkey.pem /etc/unbound/privkey.pem
COPY ./fullchain.pem /etc/unbound/fullchain.pem
COPY ./unbound.sh /unbound.sh

WORKDIR /etc/unbound/

EXPOSE 853

CMD ["/unbound.sh"]
```	
**Build Image**

	docker build -t unbound-tls:1.9.2 .

**Docker Compose**

```
vim docker-compose.yml
```

Inside, paste the following script:

```yaml
version: '3'
services:
  unbound-tls:
    image: unbound-tls:1.9.2
    container_name: unbound-tls
    restart: always
    ports:
      - "853:853"
```

**Run image**

	docker-compose up -d
