# homecloud_local
Running nextcloud + traefik + bitwarden on localhost with docker

## Prerequisites
0. Install git (bash or powershell)
1. Docker and Docker Compose installed and running on machine
2. Get an installation id and key from [bitwarden.com/host](https://bitwarden.com/host)
3. Get a Sendgrid Account at [sendgrid.com](https://sendgrid.com)
3. Create network `traefik` on docker:
```
docker network create traefik
```

---

## installation and configuration bitwarden
1. Clone files from this repo:
```
git clone https://github.com/benjaminpreiss/homecloud_local.git
```
2. Download bitwarden install script (make sure that it is downloaded to the same directory as `nextcloud-docker-compose.yml`):
    - bash:
    ```
    curl -Lso bitwarden.sh https://go.btwrdn.co/bw-sh \
    && chmod +x bitwarden.sh
    ```
    - powershell:
    ```
    Invoke-RestMethod -OutFile bitwarden.ps1 `
        -Uri https://go.btwrdn.co/bw-ps
    ```

3. Install bitwarden via script and complete Prompts in the installer. Press Enter without specifying a domain. Decline the questions regarding certificates. Enter your installation id and key.
    - bash:
    ```
    ./bitwarden.sh install
    ```
    - powershell:
    ```
    .\bitwarden.ps1 -install
    ```

4. Adjust `./bwdata/config.yml`:
    - `http_port: 8090`
    - `https_port: 8443`

5. Adjust `./bwdata/env/global.override.env` to configure SMTP mail server settings, YubiKey OTP API credentials, HaveIBeenPwned (HIBP) breach report API key, etc. Most importantly these lines (replace smtp settings with data from your sendgrid account, otherwise account creation won't work):

```
globalSettings__baseServiceUri__vault=https://bitwarden.localhost
globalSettings__baseServiceUri__api=https://bitwarden.localhost/api
globalSettings__baseServiceUri__identity=https://bitwarden.localhost/identity
globalSettings__baseServiceUri__admin=https://bitwarden.localhost/admin
globalSettings__baseServiceUri__notifications=https://bitwarden.localhost/notifications
...
globalSettings__attachment__baseUrl=https://bitwarden.localhost/attachments
...
globalSettings__mail__replyToEmail=****@****.***
globalSettings__mail__smtp__host=smtp.sendgrid.net
globalSettings__mail__smtp__port=587
globalSettings__mail__smtp__ssl=false
globalSettings__mail__smtp__username=apikey
globalSettings__mail__smtp__password=***********
```

6. Create `docker-compose.override.yml` at `./bwdata/docker/docker-compose.override.yml`:
```
version: '3'

services:
  nginx:
    labels:
      - traefik.enable=true
      - traefik.docker.network=traefik
      
      # http service and router for bitwarden
      - traefik.http.routers.bitwarden_router_http.entryPoints=web
      - traefik.http.routers.bitwarden_router_http.service=bitwarden_service_http
      - traefik.http.routers.bitwarden_router_http.rule=Host("bitwarden.localhost")
      - traefik.http.services.bitwarden_service_http.loadbalancer.server.port=8080
      
      # https service and router for bitwarden
      - traefik.http.routers.bitwarden_router_https.entryPoints=websecure
      - traefik.http.routers.bitwarden_router_https.tls=true
      - traefik.http.routers.bitwarden_router_https.service=bitwarden_service_https
      - traefik.http.routers.bitwarden_router_https.rule=Host("bitwarden.localhost")
      - "traefik.http.routers.bitwarden_router_https.tls.certresolver=myresolver"
      - traefik.http.services.bitwarden_service_https.loadbalancer.server.port=8080

      # permanent http to https redirect
      - traefik.http.routers.bitwarden_router_http.middlewares=bitwarden_https_redirect
      - traefik.http.middlewares.bitwarden_https_redirect.redirectscheme.scheme=https
      - traefik.http.middlewares.bitwarden_https_redirect.redirectscheme.permanent=true
    networks:
      - traefik
      - default

networks:
  traefik:
    external: true
```

7. Rebuild bitwarden:
    - bash:
    ```
    ./bitwarden.sh rebuild
    ```
    - powershell:
    ```
    .\bitwarden.ps1 -rebuild
    ```

## Configuration nextcloud
1. Adjust passwords and usernames in `./db.env`
2. Make sure `nextcloud-docker-compose.yml` is located at `./`

## Configuration traefik
1. Make sure `traefik-docker-compose.yml` is located at `./`

---

## Start containers
1. Start bitwarden:
    - bash:
    ```
    ./bitwarden.sh start
    ```
    - powershell:
    ```
    .\bitwarden.ps1 -start
    ```
2. Start nextcloud:
```
docker-compose -f ./nextcloud-docker-compose.yml up -d
```
3. Start traefik:
```
docker-compose -f ./traefik-docker-compose.yml up -d
```

---

## Access
Now you should be able to access 
- The traefik web UI at [localhost:8080](http://localhost:8080)
- Nextcloud at [nextcloud.localhost](http://nextcloud.localhost)
- Bitwarden at [bitwarden.localhost](http://bitwarden.localhost)

The browser will try to keep you from accessing [bitwarden.localhost](http://bitwarden.localhost) and [nextcloud.localhost](http://nextcloud.localhost), because of invalid certificates. Nevertheless you can insist on entering the page and ignore the warnings about security.

---

## Credits
Most of this I didn't find out by myself.
It is a compilation of findings from
- [dja93a on reddit](https://www.reddit.com/r/Bitwarden/comments/dja93a/selfhosted_bitwarden_behind_traefik_v20_reverse/)
- [Chris Wiegman](https://chriswiegman.com/2020/01/running-nextcloud-with-docker-and-traefik-2/)
- [Joshua Avalon](https://medium.com/@joshuaavalon/setup-traefik-v2-step-by-step-fae44ed8f76d)
- [Sam Texas](https://www.simplecto.com/traefik-2-0-docker-and-letsencrypt/)