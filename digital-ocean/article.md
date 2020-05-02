## Prep
There are no doubt infinitely more complicated and secure actions you might take to secure your services, so do your own research too! *I am not a web security expert!*. 

Set yourself up with a DNS name (try [123-Reg](https://www.123-reg.co.uk) for example), and [setup your nameserver](https://www.digitalocean.com/community/tutorials/how-to-point-to-digitalocean-nameservers-from-common-domain-registrars) to point at DigitalOcean. Then you can create your DNS configuration [within DO](https://www.digitalocean.com/docs/networking/dns/).

## Starting a Droplet
Machines in Digital Ocean are referred to as Droplets. We need to start a Droplet to start playing with the hosting. I suggest you start with an Ubuntu server droplet in the cheapest size for testing - you can always upgrade later, but if you start larger and it gets too expensive, it's more difficult to downgrade. As you add more services you may be forced to expand. 

## Stopping outside access
It's sensible to also setup a Firewall to protect your system whilst you work on it. You should only open the ports which you explicitly need. For a webserver, this is probably ports `80` and `433` only. Personally I keep a `development` firewall which acts on the corresponding tag. This opens port 22 to my home IP address, allowing me access. I remove this tag once I'm done developing. 

You should also setup UFW within your server if you're using Ubuntu as I started with. To do that I followed this guide for using [UFW to secure webservers](https://www.tecmint.com/setup-ufw-firewall-on-ubuntu-and-debian/). Again, I allow port `80` and `443`, and SSH to my home IP 

### Other things to do 
1. Setup a SSH using an SSH Key pair so you can control access. Basics:
```
ssh-keygen -t rsa
ssh-copy-id you@your-server.com
```

Once you have this working OK, you should consider turning off password access to SSH (allowing access using the key only), and not permitting Root access at any point.

You can achieve this by editing your `/etc/ssh/sshd.conf` to add lines:
```
PermitRootLogin no
PasswordAuthentication no
```
2. Setup monitoring which uses Digital Ocean alerts. This can email you when your droplet is using too much memory or CPU. 

3. Keep your server up to date! you'll need to keep track of security patches and check in often to see if your server needs rebooting to update.

## Hosting
This site is a Ghost install, running on Docker. The network looks like this:



I achieve this using three key components:
* Ghost instance
* MySQL database
* Traefik proxy

Traefik also takes care of acquiring and maintaining a Let's Encrypt SSL certificate for the site. Easy! I followed the Traefik docs to suit my needs.

```
version: "3.3"
services:

  traefik:
    image: "traefik:v2.2"
    container_name: "traefik"
    restart: "always"
    ports:
      - "80:80"
      - "443:443"

    command:

      # General setup
      - "--global.sendAnonymousUsage"
      - "--log.filePath=/var/log/traefik.log"
      - "--accesslog=true"
      - "--accesslog.filePath=/var/log/access.log"
      - "--accesslog.bufferingsize=100"

      # Provide a dashboard - but this is protected by middleware  
      - "--api.dashboard=true"

      # Use the docker provider, but don't expose by default
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--providers.file.directory=/configuration"
      - "--providers.file.filename=dynamic_conf.toml"
      - "--providers.file.watch=true"

      # Create a http and https entrypoint
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"

      # We'll store the certs in a file acme.json which we mount below
      - "--certificatesresolvers.mytlschallenge.acme.tlschallenge=true"
      - "--certificatesresolvers.mytlschallenge.acme.email=you@yourhost.net"
      - "--certificatesresolvers.mytlschallenge.acme.storage=/letsencrypt/acme.json"

    # Containers to be shared need to be on this network 
    # so Traefik can route to them. This allows you to segregate 
    # your databases etc onto a network Traefik cannot see. 
    networks:
      - web

    volumes:
      - logs:/var/log
      - "./letsencrypt:/letsencrypt"
      - "./configuration:/configuration"
      # Mounting the docker socket allows us to pick up on containers
      # being created, and automatically start serving them. 
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
volumes:
  logs: {}

networks:
  web:
     external: true
```

This then works with a `toml` file which configures more of our server. This needs to go in the `configuration` folder. You'll need to go through and setup the various parts of this file to suit your personal situation. Change the `Host` and `ipWhitelist` as you require. The `ipWhitelist` restricts access to the dashboard to your home IP. 
```toml
[http]

  [http.routers.my-api]
    entryPoints = ["websecure"]
    rule = "Host(`yourserver.net`)"
    service = "api@internal"
    middlewares = ["secured"]
    [http.routers.my-api.tls]
      options = "default"
      certResolver = "mytlschallenge"

  [http.middlewares]
 
    [http.middlewares.secured.chain]
      middlewares = ["safe-ipwhitelist", "auth"]

    [http.middlewares.secureHeader.headers]
      frameDeny = true
      sslRedirect = true
      stsSeconds = 31536000
      stsPreload = true
      stsIncludeSubdomains = true
  
    [http.middlewares.auth.basicAuth]
      users = [
      "user1:some-special-password"
      ]

    [http.middlewares.safe-ipwhitelist.ipWhiteList]
      sourceRange = ["192.168.1.1"]

[tls]

  [tls.options]
    [tls.options.default]
      minVersion = "VersionTLS12"
      cipherSuites = [
        "TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384",
        "TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305",
        "TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305",
        "TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256",
        "TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256",
      ]
      sniStrict = true

```

And ghost is done as follows:
```
version: "3"

services:
   ghost:
     image: ghost:3.14-alpine
     volumes:
        - ./volumes/ghost:/var/lib/ghost/content
     logging:
        driver: "json-file"
        options:
            max-file: "5"
            max-size: "10m"
     networks:
        - web 
        - ghost_back 
     restart: always
     labels:
        - "traefik.enable=true"
        - "traefik.docker.network=web"
        - "traefik.http.middlewares.yourwebsite-https.redirectscheme.scheme=https"

        # Handle HTTP traffic by redirecting to the HTTPS endpoint
        - "traefik.http.routers.yourwebsite-http.entrypoints=web"
        - "traefik.http.routers.yourwebsite-http.rule=Host(`yourwebsite.net`, `www.yourwebsite.net`)"
        - "traefik.http.routers.yourwebsite-http.middlewares=yourwebsite-https@docker"
        
        # Switch these lines to restrict access to the whitelist, or use in production
        - "traefik.http.routers.yourwebsite.middlewares=secureHeader@file"
        #- "traefik.http.routers.yourwebsite.middlewares=safe-ipwhitelist@file"
        
        # This router handles https traffic:
        - "traefik.http.routers.yourwebsite.entrypoints=websecure"
        - "traefik.http.routers.yourwebsite.rule=Host(`yourwebsite.net`, `www.yourwebsite.net`)"
        - "traefik.http.routers.yourwebsite.tls=true"
        - "traefik.http.routers.yourwebsite.tls.options=default"
        - "traefik.http.routers.yourwebsite.tls.certresolver=mytlschallenge"
     environment:
       database__client: mysql
       database__connection__host: ghost_mysql
       database__connection__user: ghost
       database__connection__password: ghostdbpass
       database__connection__database: ghostdb_yourwebsite_net
       url: https://www.yourwebsite.net
     container_name: ghost_yourwebsite_net 
networks:
  # The ghost backbone handles the connection to the 
  # database
  ghost_back:
     external: true
  
  # The web service allows Traefik to see us 
  web:
     external: true

```

Then it's just a case of bringing everything up using `docker-compose up`!