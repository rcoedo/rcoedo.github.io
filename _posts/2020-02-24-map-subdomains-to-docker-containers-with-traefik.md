---
layout: post
resources: map-subdomains-to-docker-containers-with-traefik
title: Map Subdomains to Docker Containers with Traefik
date: 2020-02-24
footer_content: This article was [originally written for the Trabe medium publication](https://medium.com/trabe/continuation-local-storage-for-easy-context-passing-in-node-js-2461c2120284), a collection of excellent articles written by [the awesome people from trabe.io](https://trabe.io/).
---

I run a server at home where I host some services for myself. I run a calibre-web instance, a transmission instance, and a plex server, among others.

I've wanted to share access to these services with my family for some time but accessing the services via `IP:PORT` URLs is a bit harsh for non-tech users.

We are going to solve that by using a reverse proxy.

## Mapping our services using a reverse proxy

A reverse proxy is a piece of software that listens for requests at some port, parses them, and fetches data from other servers based on the parsed information.

This is easier to understand with an example. Let's say we have our calibre service running at `localhost:9083` and our transmission server at `localhost:9091.` We want `books.mydomain.com` pointing to `localhost:9083` and `torrents.mydomain.com` pointing to `localhost:9091.` What our reverse proxy has to do in this case is; take the request and, based on the subdomain portion, forward the request to the appropriate service.

## Using Traefik as our reverse proxy

[Traefik](https://doc.traefik.io/traefik/) is a reverse proxy with auto-discoverability. It can analyze your network to determine which service should handle a request.

Traefik defines some abstractions that are used to configure your network:
Providers: Providers are used to discovering what services are living in your infrastructure. There are providers for Docker, Kubernetes, or file configuration.
Entry points: Entry points are the gates to Traefik. You have to define what ports Traefik is going to listen to.
Routers: Routers take the request from an entry point and forward the request to a service based on a set of rules.
Middleware: Middleware is chainable components that can be used in routers. It can add headers to the original request, do authentication, etc.
Services: These are the definitions of the services that will actually handle the request. These would be calibre, transmission, or plex in our example.

## Our example scenario

First, we define a `docker-compose.yml` file for our Traefik service, and pass some flags to it:

```yaml
version: '3'
services:
  traefik:
    image: traefik:v2.0
    container_name: traefik
    command:
      - '--api.insecure=true'
      - '--providers.docker=true'
      - '--providers.docker.exposedbydefault=false'
      - '--entrypoints.web.address=:80'
    restart: unless-stopped
    ports:
      - '80:80'
      - '8080:8080'
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
```

The flags define our provider, in this case our provider is docker. Using this option, Traefik will watch our docker host and update the services when they appear. We also need to mount the host docker socket to make this work.

We defined our entry point here. Traefik will listen to incoming HTTP requests on port 80.

## Adding Traefik's labels to our existing containers

To enable Traefik to discover our services, all we have to do now is to add labels to them. In their `docker-compose.yml` definition, we add this to their labels:

```yaml
version: '3'
services:
  calibre:
    image: linuxserver/calibre-web
    container_name: calibre
    restart: unless-stopped
    ports:
      - '9083:8083'
    labels:
      - 'traefik.enable=true'
      - 'traefik.http.routers.cal.rule=Host(`books.mydomain.com`)'
      - 'traefik.http.routers.cal.entrypoints=web'
      - 'traefik.http.services.cal.loadbalancer.server.port=9083'
```

We added a new router `cal` connected to the web entry point, which will use the `Host('books.mydomain.com')` rule. This rule matches the URL and maps the request to port 9083.

We add these labels to every docker container, and we're good to go!

## Wrapping it up

If you need a reverse proxy, Traefik is excellent and easier to configure than other classic tools like [pound](<https://en.wikipedia.org/wiki/Pound_(networking)>). Traefik can also do [load balancing](https://doc.traefik.io/traefik/routing/services/#servers-load-balancer), and [you can even configure it to generate HTTPS certificates automatically](https://doc.traefik.io/traefik/https/acme/) using [let's encrypt](https://letsencrypt.org/).

I hope you enjoyed this post and see you in another one!
