# homecloud_local
Running nextcloud + traefik + bitwarden on localhost with docker

## Prerequisites
0. Install git (bash or powershell)
1. Docker and Docker Compose installed and running on machine
2. Get an installation id and key from [https://bitwarden.com/host](https://bitwarden.com/host)
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

3. Install bitwarden via script and complete Prompts in the installer. For the questions regarding certificates just press enter.
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

5. Adjust `./bwdata/env/global.override.env` to configure SMTP mail server settings, YubiKey OTP API credentials, HaveIBeenPwned (HIBP) breach report API key, etc.

6. Create `docker-compose.override.yml` at `./bwdata/docker/docker-compose.override.yml`:
```
version: '3'

services:
  nginx:
    labels:
      - traefik.enable=true
      - traefik.docker.network=traefik
      - traefik.backend=bitwarden-nginx
      - traefik.http.routers.bitwarden.entryPoints=web
      - traefik.http.routers.bitwarden.rule=Host("bitwarden.localhost")
      - traefik.http.services.bitwarden.loadbalancer.server.port=8090
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
docker compose -f ./nextcloud-docker-compose.yml up -d
```
3. Start traefik:
```
docker compose -f ./traefik-docker-compose.yml up -d
```

---

## Access
Now you should be able to access 
- The traefik web UI at [http://localhost:8080](http://localhost:8080)
- Nextcloud at [http://nextcloud.localhost](http://nextcloud.localhost)
- Bitwarden at [http://bitwarden.localhost](http://bitwarden.localhost)