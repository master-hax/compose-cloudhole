# compose-cloudhole

a multi-container Docker application to run a [pi-hole](https://hub.docker.com/r/pihole/pihole) ad-blocking DNS server that uses a [cloudflared](https://hub.docker.com/r/visibilityspots/cloudflared) DNS over HTTPS proxy as the upstream resolver

## how to set it up

1. download [docker-compose.yml](/docker-compose.yml)
1. create a file named `pwd` containing the desired password for the web interface
1. run `docker-compose up`

if everything works correctly, you should be able to resolve hosts securely & without ads on port 53!

## how to use it

the DNS resolver should be accessible at `0.0.0.0:53`

the pi-hole web UI should be accessible at http://localhost:54/admin

### to use cloudhole for all containers:
set Docker host to use `127.0.0.1` as the DNS resolver. all your containers will use pi-hole automatically through the Docker engine.

**note**: don't set the Docker host to use its own external/LAN IP address as the resolver (as you might through DHCP), see below

### to override a specific container with cloudhole:
```
services:
 application:
  dns:
   - 172.16.1.4 # cloudhole's local IP
```

### to override a specific container with an arbitrary DNS resolver:
```
services:
 application:
  dns:
   - 127.0.0.11 # built in docker engine DNS proxy, uses host's resolver
   - 8.8.8.8    # google plaintext DNS
   - 1.1.1.1    # cloudflare plaintext DNS
```

to use cloudhole across all devices on your network, give your Docker host a static IP address & advertise it using your router's DHCP server. you can go one step further and set up firewalls to block all outbound (UDP) traffic to port 53 as the DNS traffic from cloudflared will manifest as encrypted TCP traffic to port 443.

### note for Docker hosts using DHCP:

if your Docker host is set to use its own external IP address as the DNS resolver (that it might have gotten via DHCP from your router), this will break DNS resolution for your containers. the reason for this is that the DNS requests that are addressed externally will reach the pi-hole but the packets won't reach the containers on their way back.

however using a static `127.0.0.1` on the host or `172.16.1.4` in the container works fine & the Docker engine is able to route the responses successfully.

## how it works

the `dns-server` service (pi-hole) & the `doh-client` service (cloudflared) have static IP addresses on a local docker network, which allows them to communicate without DNS resolution being available. port 53 on the host is mapped to `dns-service` to allow it to receive requests from the outside network. pi-hole blocks some of these requests if they are for ad servers, & responds to some of them from its cache.

the rest of the requests are then sent to the upstream DNS resolver, `doh-client`, which sends them out of your network to `1.1.1.1` over [an encrypted HTTPS tunnel](https://en.wikipedia.org/wiki/DNS_over_HTTPS) rather than the default plaintext UDP packets.
