+++
title = 'Spinning up a Dockerized Onion Mirror'
date = 2024-04-07T20:21:17+03:00
author = ''
draft = false
tags = ['tor', 'docker', 'self-hosting', 'privacy']
categories = []
+++

I decided to spin up an [onion mirror of this website](http://golfed6fzytoktol4de4o4nerap3xuykhfm5makfzscib65df3khnpyd.onion) just for the fun of it. Funnily enough hosting an onion service is actually easier than hosting a clearweb site.

When searching for information about Dockerizing onion services, I noticed that the guides found with quick web searches vary significantly in quality, especially from a security standpoint. This prompted me to compile my own notes and thoughts on the topic into this compact post.

## TL;DR

If you only want to use Tor as a rewriting proxy (i.e. client types in the v3 address, and the proxy serves the upstream clearweb site through Tor), [Onionspray](https://gitlab.torproject.org/tpo/onion-services/onionspray/) is a great choice.

For those who aren't concerned about fine-tuning or more in-depth details of the configuration, here's [a great repository to get started with](https://github.com/ha1fdan/hidden-service-docker) that I also used as the base for my setup. The compose configuration found in the repository manually installs the newest available version of Tor, which is much better than relying on the image's contributors to update it in a premade image.

If you want a vanity v3 address, you can use a tool like [mkp2240](https://github.com/cathugger/mkp224o).

## Setup

Here's the slightly modified `docker-compose.yml` I use:

```yaml
services:
  tor:
    image: alpine:latest
    container_name: tor
    command: sh -c "apk update && apk add --no-cache tor && chmod 700 /var/lib/tor/onion-service && (cat /var/lib/tor/onion-service/hostname || echo 'Hostname not available.') && tor -f /etc/tor/torrc"
    volumes:
      - ./tor:/etc/tor:rw
      - ./onion-mirror:/var/lib/tor/onion-service:rw
      - nginx-tor-socket:/var/run/onion-sockets:rw
    depends_on:
      - nginx
    restart: unless-stopped

  nginx:
    image: nginx:latest
    container_name: nginx-onion
    command: 
    volumes:
      - ./web/config/onion-default.conf:/etc/nginx/conf.d/default.conf:rw
      - ./web/public:/usr/share/nginx/html:ro
      - nginx-tor-socket:/var/run/onion-sockets:rw
    healthcheck:
      test: ["CMD", "curl", "-f", "--unix-socket", "/var/run/onion-sockets/site.sock", "||", "exit 1"]
      interval: 10s
      timeout: 5s
      retries: 5
    restart: unless-stopped

volumes:
  nginx-tor-socket:
```

Combined with a minimal nginx configuration and `torrc`:

```text
server {
    listen unix:/var/run/onion-sockets/site.sock;
    server_name golfed6fzytoktol4de4o4nerap3xuykhfm5makfzscib65df3khnpyd.onion;

    access_log off;
    server_tokens off;

    add_header X-Content-Type-Options "nosniff";
    add_header X-Frame-Options SAMEORIGIN;
    proxy_hide_header X-Powered-By;

    location / {
        root /usr/share/nginx/html;
        index index.html index.htm;
    }
}
```

```text
HiddenServiceDir /var/lib/tor/onion-service/
HiddenServicePort 80 unix:/var/run/onion-sockets/site.sock
```

## In-depth configuration

Despite my use case being quite casual, I still wanted to follow [the best practices of hosting an onion service](https://riseup.net/en/security/network-security/tor/onionservices-best-practices).

### Service isolation

As recommended in the Riseup guide, it's crucial to carefully isolate clearweb services from the onion ones to prevent any unwanted information leaks. For more critical services, it's definitely worth hosting them in a completely different location with only a minimal number of public-facing ports open. Personally, I solved this by spinning up an individual nginx container that essentially hosts a separate copy of this site.

### Sockets over TCP

Since Tor doesn't require binding to physical ports, there's no need to worry about conflicting ports. Instead, it's worth to consider using Unix sockets for local communication rather than TCP. Unix sockets can reduce the overhead of TCP/IP networking by allowing local inter-process communication to occur through the file system. They can also offer a bit more security by not accidentally exposing services through network ports (although this is already quite easily manageable with containers as long as you don't mix up the ports of different services).

```text
HiddenServicePort 80 unix:/var/run/onion-sockets/site.sock
```

```text
server {
    listen unix:/var/run/onion-sockets/site.sock;
    server_name golfed6fzytoktol4de4o4nerap3xuykhfm5makfzscib65df3khnpyd.onion;
}
```

### Do you need TLS?

Most of the time, there's no need for a TLS certificate with an onion service. As [this section](https://community.torproject.org/onion-services/advanced/https/) well describes, you might lose more than you gain: many CAs don't support the .onion TLD, and third-party certificates might unintentionally leak .onion names. Fortunately, you can now get DV certificates (instead of EV certificates) for onion sites from the Greek non-profit HARICA. However, support from Let's Encrypt CA is still missing.

The few benefits of having a certificate include using the `https` URI scheme and ensuring the traffic from your web server to Tor is encrypted (which could be especially useful if the instances aren't running on the same machine).

### Onionscan

[Onionscan](https://onionscan.org/) is the Tor Project's contributors' recommendation for detecting possible misconfigurations or information leaks. The original project is practically abandoned, but here's [an up-to-date fork](https://github.com/415ALS/onionscanv3) with v3 address support.

### Onion-Location

To take advantage of Tor Browser's Onion-Location redirection, you should add the following response header to your clearweb site's configuration:

```text
add_header Onion-Location http://golfed6fzytoktol4de4o4nerap3xuykhfm5makfzscib65df3khnpyd.onion$request_uri;
```

Or alternatively include an HTML `<meta>` attribute:

```html
<meta http-equiv="onion-location" content="http://golfed6fzytoktol4de4o4nerap3xuykhfm5makfzscib65df3khnpyd.onion" />
```

Notably, with proxying enabled through Cloudflare, I encountered difficulties in getting the response headers to pass through to the client, necessitating the use of the `<meta>` attribute instead.

### Something else?

If you notice any critical or less critical details missing, please inform me [via email or XMPP](/contact.txt). I purposely didn't delve into load balancing (with Onionbalance) or Vanguards as they're a bit too broad areas to cover for this post, and I'm not very familiar with them to begin with.

## Sources

I highly recommend checking out the sites I browsed while figuring this stuff out, especially [The Onion Services Ecosystem](https://tpo.pages.torproject.net/onion-services/ecosystem/):

- [Tor Project's Guide on Setting Up Onion Service](https://community.torproject.org/onion-services/setup/)
- [Tor Project's Tips on Operational Security](https://community.torproject.org/onion-services/advanced/opsec/)
- [Tor Project's Tips on HTTPS for Onion Services](https://community.torproject.org/onion-services/advanced/https/)
- [Riseup's Best Practices Guide](https://riseup.net/en/security/network-security/tor/onionservices-best-practices)
- [Introduction to Onionspray](https://tpo.pages.torproject.net/onion-services/ecosystem/apps/web/onionspray/)
- [Tor Project's Onionsite Checklist](https://tpo.pages.torproject.net/onion-services/ecosystem/apps/web/checklist/)
- ["Connect two NGINX's through UNIX sockets" by David Sierra](https://blog.davidsierra.dev/posts/connect-nginxs-through-sockets/)
- ["Create a complete Tor Onion Service with Docker and OpenSUSE in less than 15 minutes" by Jason S. Evans](https://www.youtube.com/watch?v=iUxiTk6w1sc)
- [Onionscan Documentation](https://onionscan.org/)

