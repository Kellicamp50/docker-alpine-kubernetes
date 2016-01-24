
# Alpine-Kubernetes base image

[![CircleCI](https://img.shields.io/circleci/project/janeczku/docker-alpine-kubernetes.svg?style=flat-square)](https://circleci.com/gh/janeczku/docker-alpine-kubernetes)

The Alpine-Kubernetes base image enables deployment of Alpine Linux micro-service containers in Kubernetes, Consul, Tutum or other Docker cluster environments that use DNS-based service discovery and rely on the containers being able to use the `search` keyword from `resolv.conf` to resolve service names.

## Supported tags and respective `Dockerfile` links

-	[`3.2` (*versions/3.2/Dockerfile*)](versions/3.2/Dockerfile)
-	[`3.3`, `latest` (*versions/3.3/Dockerfile*)](versions/3.3/Dockerfile)

-------
[![Imagelayers](https://badge.imagelayers.io/janeczku/alpine-kubernetes:3.3.svg)](https://imagelayers.io/?images=janeczku/alpine-kubernetes:3.3 'Get your own badge on imagelayers.io')
[![Docker Pulls](https://img.shields.io/docker/pulls/janeczku/alpine-kubernetes.svg?style=flat-square)](https://hub.docker.com/r/janeczku/alpine-kubernetes/)

## About

Alpine-Kubernetes is derived from the official [Docker Alpine](https://hub.docker.com/_/alpine/) image adding the excellent [s6 supervisor for containers](https://github.com/just-containers/s6-overlay) and [go-dnsmasq DNS server](https://github.com/janeczku/go-dnsmasq). Both s6 overlay and go-dnsmasq introduce very minimal runtime and filesystem overhead.    
Trusted builds are available on [Docker Hub](https://hub.docker.com/r/janeczku/alpine-kubernetes/).

## Motivation
Alpine Linux does not support the `search` keyword in resolv.conf. This breaks many tools that rely on DNS service discovery (e.g. Kubernetes, Tutum.co, Consul).
Additionally Alpine Linux deviates from the well established pardigm of always querying the primary DNS server first. This introduces problems in cases where the container has multiple nameserver with inconsistent records (e.g. one Consul server and one recursing server).
    
To overcome these issues the Alpine-Kubernetes base image includes a lightweight (1.2 MB) container-only DNS server that replicates the behavior of GNU libc's stub-resolver.

## How it works
On container start the embedded DNS server parses the `nameserver` and `search` entries from /etc/resolv.conf and configures itself as primary nameserver for the container. DNS queries from local processes are answered according to the following conventions:
* The nameserver listed first in resolv.conf is always queried first. Additional nameservers are treated as fallbacks.
* Hostnames are qualified by appending the domains configured with the `search` keyword in resolv.conf
* Single-label hostnames (e.g.: "redis-master") are always qualified with search domains
* Multi-label hostnames are first tried as absolute names and only then qualified with search domains

## Usage

Building your own image based on Alpine-Kubernetes is as easy as typing `FROM janeczku/alpine-kubernetes`.    
The official Alpine Docker image is well documented, so check out [their documentation](http://gliderlabs.viewdocs.io/docker-alpine) to learn more about building micro Docker images with Alpine Linux.

*The small print:*    
Do NOT redeclare the `ENTRYPOINT` in your Dockerfile as this is reserved for the supervisor init script.

### Example Alpine Redis image

```Dockerfile
FROM janeczku/alpine-kubernetes:3.3
RUN apk-install redis
CMD ["redis-server"]
```

### Multiple processes in a single container (optional)

You can leverage s6 supervised services to run multiple processes in a single container. Instructions can be found [here](https://github.com/just-containers/s6-overlay#writing-a-service-script). Since the container DNS server itself is a service, any additional services need to be configured to start **after** the DNS service. This is accomplished by adding the following line to the service script:

> if { s6-svwait -t 5000 -u /var/run/s6/services/resolver }

#### Example service script

```BASH
#!/usr/bin/execlineb -P
if { s6-svwait -t 5000 -u /var/run/s6/services/resolver }
with-contenv
nginx
```

## Docker Hub image tags

Alpine-Kubernetes image tags follow the official [Alpine Linux image](https://hub.docker.com/_/alpine/). See the top of this page for the currently available versions.
To build your images with the latest version of Alpine-Kubernetes that is based on Alpine Linux 3.3 use: 

```Dockerfile
FROM janeczku/alpine-kubernetes:3.3
```

### Container DNS server configuration (optional)
The configuration of the go-dnsmasq DNS server can be changed by setting environment variables either at runtime with `docker run -e ...` or in the Dockerfile.
Check out the [documentation](https://github.com/janeczku/go-dnsmasq) for the available configuration options.

## Acknowledgement

* [Gliderlabs](http://gliderlabs.com/) for providing the official [Alpine Docker image](https://hub.docker.com/_/alpine/)
* [Sillien](http://gliderlabs.com/) for coming up with the original idea of creating a base image dealing with Alpine Linux's DNS shortcomings in Tutum/Kubernets clusters: [base-alpine](https://github.com/sillelien/base-alpine/)

