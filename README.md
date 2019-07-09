# Table of Contents

- [Introduction](#introduction)
- [Usage](#usage)
- [New features](#new-features)
  - [--proxy parameter](#proxy-parameter)
  - [--resolve-to-proxy-ip-env parameter](#resolve-to-proxy-ip-env-parameter)
    - [HTTP container](#http-container)
    - [Gitlab](#gitlab)
  - [--additional-dns-names-env parameter](#additional-dns-names-env-parameter)
    - [Example of multiple DNS names for a single container](#example-multiple-dns)
  - [--proxy-network parameter](#proxy-network-parameter)
- [Credits](#credits)
- [License](#license)

# Introduction

Docker container that resolves container FQDN to IPs. Intended to be used with a reverse proxy, such as nginx.

# Usage

Start the docker container:

```
docker run \
 --name dns \
 -p 53:53/udp \
 -v /var/run/docker.sock:/docker.sock:ro \
 -d kedu/dns \
 --domain example.com \
 --proxy nginx \
 --resolve-to-proxy-ip-env RESOLVE_TO_PROXY_IP \
 --proxy-network network-proxy
```

Run some containers:

```
    % docker run -d --name foo ubuntu bash -c "sleep 600"
```

Start up dockerdns:

```
    % docker run --name dns -v /var/run/docker.sock:/docker.sock jamgocoop/docker-dns \
        --domain example.com
```

Start more containers:

```
    % docker run -d --name bar ubuntu bash -c "sleep 600"
```

Check dockerdns logs:

```
    % docker logs dns
    2014-10-08T20:45:37.349161 [dockerdns] table.add dns.example.com -> 172.17.0.3
    2014-10-08T20:45:37.351574 [dockerdns] table.add foo.example.com -> 172.17.0.2
    2014-10-08T20:45:37.351574 [dockerdns] table.add bar.example.com -> 172.17.0.4
```

Start up dockerdns with multiple domains:

```
    % docker run --name dns -v /var/run/docker.sock:/docker.sock jamgocoop/docker-dns \
        --domain example.com alias.com
```

Check dockerdns logs:

```
    % docker logs dns
    2014-10-08T20:45:37.349161 [dockerdns] table.add dns.example.com -> 172.17.0.3
    2014-10-08T20:45:37.349161 [dockerdns] table.add dns.alias.com -> 172.17.0.3
    2014-10-08T20:45:37.351574 [dockerdns] table.add foo.example.com -> 172.17.0.2
    2014-10-08T20:45:37.351574 [dockerdns] table.add foo.alias.com -> 172.17.0.2
    2014-10-08T20:45:37.351574 [dockerdns] table.add bar.example.com -> 172.17.0.4
    2014-10-08T20:45:37.351574 [dockerdns] table.add bar.alias.com -> 172.17.0.4
```

Query for the containers by hostname:

```
    % host foo.example.com 172.17.0.3
    Using domain server:
    Name: 172.17.0.3
    Address: 172.17.0.3#53
    Aliases:

    foo.example.com has address 172.17.0.2
    foo.example.com has address 172.17.0.2
```

Use dns container as a resolver inside a container:

```
    % docker run -it --dns $(docker inspect -f '{{.NetworkSettings.IPAddress}}' dns) \
        --dns-search example.com ubuntu bash

    root@95840788bf08:/# cat /etc/resolv.conf
    nameserver 172.17.0.3
    search example.com

    root@95840788bf08:/# ping foo
    PING foo.example.com (172.17.0.2) 56(84) bytes of data.
    64 bytes from 172.17.0.2: icmp_seq=1 ttl=64 time=0.112 ms
    64 bytes from 172.17.0.2: icmp_seq=2 ttl=64 time=0.112 ms
```

Names not rooted in `example.com` will be resolved recursively using Google's resolver `8.8.8.8` by default:

```
    root@95840788bf08:/# ping github.com
    PING github.com (192.30.252.128) 56(84) bytes of data.
    64 bytes from 192.30.252.128: icmp_seq=1 ttl=61 time=21.3 ms
```

To disable recursive resolution, use the `--no-recursion` flag:

```
    % docker run --name dns -v /var/run/docker.sock:/docker.sock jamgocoop/docker-dns \
        --domain example.com --no-recursion
```

Now names not rooted in `example.com` will fail to resolve:

```
    % docker run -it --dns $(docker inspect -f '{{.NetworkSettings.IPAddress}}' dns) \
        --dns-search example.com ubuntu bash
    root@4d15342387b0:/# ping github.com
    ping: unknown host github.com
```

To add static host name, use the `--record` option:

```
    % docker run --name dns -v /var/run/docker.sock:/docker.sock jamgocoop/docker-dns \
        --domain example.com --record test:172.17.0.100

	$ nslookup test.example.com 172.17.0.3 (dns container ip)
	Server:		172.17.0.3
	Address:	172.17.0.3#53

	Name:	test.example.com
	Address: 172.17.0.100
```

To add external static host name, use the `--record-external` option:

```
    % docker run --name dns -v /var/run/docker.sock:/docker.sock jamgocoop/docker-dns \
        --domain example.com --record-external www.montoto.org:10.0.0.5

	$ nslookup www.montoto.org 172.17.0.3 (dns container ip)
	Server:		172.17.0.3
	Address:	172.17.0.3#53

	Name:	www.montoto.org
	Address: 10.0.0.5

	$ nslookup www.montoto.org 8.8.8.8
	Server:		8.8.8.8
	Address:	8.8.8.8#53

	Non-authoritative answer:
	Name:	www.montoto.org
	Address: 212.89.28.201
```
# New features

Below explained the new features implemented modifying the original script

## --proxy parameter

This parameter specifies the name of the container running nginx-proxy (https://hub.docker.com/r/jwilder/nginx-proxy/) container.

It will be used in conjuntion with "--resolve-to-proxy-ip-env" parameter, explained later on

## --resolve-to-proxy-ip-env parameter

Specifies the docker container environment variable to be evaluated on each running container.

In case of detect it, the DNS will assign to the container the IP of the container with the name of the variable "--proxy"

Let's see couple of examples:

### HTTP container

We want the container to be accessible internally and externally through the proxy, so we can, for instance, use valid SSL certificates.

We should initialize the container with the "--proxy" flag pointing to  the container running nginx-proxy image and "--resolve-to-proxy-ip-env" indicating which container environment variable will be evaluated:

```
docker run \
 --name dns \
 -p 53:53/udp \
 -v /var/run/docker.sock:/docker.sock \
 -d keducoop/dns:v2 \
 --domain example.com,another.example.com \
 --proxy nginx-proxy \
 --resolve-to-proxy-ip-env RESOLVE_TO_PROXY_IP \
 --additional-dns-names-env ADDITIONAL_DNS_NAMES
```

Now we start a container with "RESOLVE_TO_PROXY_IP" environment variable:

```
docker run \
 --name httpd \
 -e RESOLVE_TO_PROXY_IP=true \
 -d httpd:2.4
```

In this example DNS container will resolve "httpd.example.com" to the same IP of the container "nginx-proxy"

### Gitlab

The gitlab container is quite special, since it's running more than one service. We would like to expose the HTTPS, but we need to access it through SSH as well.

We need to start the container **without** the **"RESOLVE_TO_PROXY_IP"** environment variable.

Example:

```
docker run --name gitlab \
 --hostname gitlab.example.com \
 -e VIRTUAL_HOST=gitlab.example.com \
 -e VIRTUAL_PORT=443 \
 -e VIRTUAL_PROTO=https \
 -e LETSENCRYPT_HOST=gitlab.example.com \
 -e LETSENCRYPT_EMAIL=user@example.com \
 --link ldap:ldap \
 -v /srv/data/computer/docker/gitlab/conf:/etc/gitlab \
 -v /srv/data/computer/docker/gitlab/data:/var/opt/gitlab \
 -v /srv/data/computer/docker/gitlab/log:/var/log/gitlab \
 -v /srv/data/computer/docker/nginx-letsencrypt/certs:/etc/gitlab/ssl:ro \
 -d gitlab/gitlab-ce:latest
```

## --additional-dns-names-env parameter

**CAUTION** this is definetely not a good solution, because it can overwrite already existing DNS entries

It's a list of comma separated values of DNS names (not FQDNs), which will be added to domains to return the FQDN.

This feature was needed by koha docker container, which exposes two virtualhosts, so there was two approaches:

* Configure (supported by koha) two different ports for each virtualhost. At the time of writing this documentation this is not supported (https://hub.docker.com/r/jwilder/nginx-proxy/). 
* Use just one port, but then the same container should have a DNS entry for each virtualhost. This is the implemented solution

### Example of multiple DNS names for a single container

First start the docker dns container with the "--additional-dns-names-env" parameter

```
docker run \
 --name dns \
 -p 53:53/udp \
 -v /var/run/docker.sock:/docker.sock \
 -d keducoop/dns:v2 \
 --domain example.com,another.example.com \
 --proxy nginx-proxy \
 --resolve-to-proxy-ip-env RESOLVE_TO_PROXY_IP \
 --additional-dns-names-env ADDITIONAL_DNS_NAMES
```

Now we start a container with "ADDITIONAL_DNS_NAMES" environment variable:

```
docker run \
 --name httpd \
 -e ADDITIONAL_DNS_NAMES=namea,nameb.admin \
 -d httpd:2.4
```

In this example DNS container will resolve all of following FQDNs to the IP of container "httpd":

(Implemented by '--domain' parameter)
* httpd.example.com
* httpd.another.example.com

(Implemented by '--additional-dns-names-env' parameter)

* namea.example.com
* namea.another.example.com
* nameb.admin.example.com
* nameb.admin.another.example.com

## --proxy-network parameter

Docker network which proxy container is attached to.

It is expected that containers whith '--resolve-to-proxy-ip-env' docker environment variable set up are attached to this network as well.

Example:

```
 --proxy-network network-proxy
```

# Credits

Taken from [docker-dns](https://github.com/phensley/docker-dns/blob/master/dockerdns) and modified to implement some features.

# License

```
    Copyright (c) 2014 Patrick Hensley

    Licensed under the Apache License, Version 2.0 (the "License");
    you may not use this file except in compliance with the License.
    You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing, software
    distributed under the License is distributed on an "AS IS" BASIS,
    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    See the License for the specific language governing permissions and
    limitations under the License.
```

