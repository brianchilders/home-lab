# Home Lab

## core - i5 Intel Nuc - Ubuntu 24.04.4 LTS
```
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sda           8:0    0  3.6T  0 disk
└─sda1        8:1    0  3.6T  0 part /srv
nvme0n1     259:0    0  1.8T  0 disk
├─nvme0n1p1 259:1    0    1G  0 part /boot/efi
└─nvme0n1p2 259:2    0  1.8T  0 part /
```


## Platform Bootstrap

### Folder Structure
```sudo mkdir -p /srv/homelab/{compose,control,data,logs,backups}
sudo chown -R $USER:$USER /srv/homelab

mkdir -p /srv/homelab/control/{caddy,openwebui,searxng,redis}
mkdir -p /srv/homelab/data/{postgres,qdrant,artifacts}
```

### Remove snapd garbage
```
snap list
sudo systemctl stop snapd
sudo systemctl disable snapd
sudo systemctl mask snapd
sudo apt purge snapd -y
sudo rm -rf /snap
sudo rm -rf /var/snap
sudo rm -rf /var/lib/snapd
sudo rm -rf ~/snap
sudo apt-mark hold snapd
which snap
systemctl list-units | grep snap
sudo systemctl disable lxd-installer.socket
sudo systemctl stop lxd-installer.socket
sudo apt purge lxd-installer -y
sudo rm -f /etc/systemd/system/snapd.mounts-pre.target
sudo systemctl daemon-reload
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl reset-failed
sudo apt autoremove -y
sudo shutdown -r now
```

### System update
```
sudo apt update
sudo apt upgrade -y
sudo shutdown -r now
```

### Docker install
```
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker
sudo shutdown -r now
docker run hello-world
```

## Services
### Open WebUI
#### Make folder
```
mkdir -p /srv/homelab/control/openwebui
```

#### .env
```
openssl rand -hex 32
nano /srv/homelab/control/openwebui/.env
```

```
OLLAMA_BASE_URL=http://<OLLAMA_IP>:11434
WEBUI_SECRET_KEY=<PASTE_GENERATED_OPENSSL_KEY_HERE>
```

#### docker-compose.yml
```
nano /srv/homelab/control/openwebui/docker-compose.yml
```

```
services:
  openwebui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: openwebui
    restart: unless-stopped
    ports:
      - "3000:8080"
    env_file:
      - .env
    volumes:
      - /srv/homelab/control/openwebui/data:/app/backend/data
    networks:
      - ai-internal

networks:
  ai-internal:
    external: true
```
#### Start Open WebUI
```
cd /srv/homelab/control/openwebui/
docker compose up -d
```

### SearXNG
#### Make folders
```
cd /srv/homelab/control/searxng
mkdir -p /srv/homelab/control/searxng/{config,redis}
```
#### docker-compose.yml
```
nano /srv/homelab/control/searxng/docker-compose.yml
```

```
services:
  redis:
    image: docker.io/valkey/valkey:8-alpine
    container_name: searxng-redis
    restart: unless-stopped
    command: valkey-server --save 30 1 --loglevel warning
    volumes:
      - /srv/homelab/control/searxng/redis:/data
    networks:
      - ai-internal

  searxng:
    image: docker.io/searxng/searxng:latest
    container_name: searxng
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      - SEARXNG_BASE_URL=http://core:8080/
      - UWSGI_WORKERS=2
      - UWSGI_THREADS=2
    volumes:
      - /srv/homelab/control/searxng/config:/etc/searxng
    networks:
      - ai-internal
    depends_on:
      - redis

networks:
  ai-internal:
    external: true
```
#### settings.yml
```
openssl rand -hex 32
nano /srv/homelab/control/searxng/config/settings.yml
```

```
use_default_settings: true

general:
  instance_name: core-searxng

search:
  formats:
    - html
    - json
  language: en-US
  region: US
  safe_search: 2

use_default_settings:
  engines:
    keep_only:
      - duckduckgo
      - brave
      - wikipedia

server:
  secret_key: <PASTE_GENERATED_OPENSSL_KEY_HERE>
  limiter: false
  image_proxy: true
  default_http_headers:
    User-Agent:
      - Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 Chrome/122.0.0.0 Safari/537.36
      - Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 Version/17.0 Safari/605.1.15

ui:
  default_settings:
    safesearch: 2

valkey:
  url: redis://redis:6379/0
```
#### Start SearXNG
```
cd /srv/homelab/control/searxng
docker compose up -d
```
#### Test SearXNG
```
curl "http://localhost:8080/search?q=test&format=json"
docker exec -it openwebui curl "http://searxng:8080/search?q=test&format=json"
```
#### Config Open WebUI:
```
Admin Panel --> Settings --> Web Search
Enable Web Search
Set Web Search Engine to searxng
Query URL: http://searxng:8080/search?q=<query>
```
### Caddy
#### Add IP to your hosts
```
<core IP> openwebui.lab
<core IP> search.lab
```
#### Make folder
```
mkdir -p /srv/homelab/control/caddy
cd /srv/homelab/control/caddy
```
#### Caddyfile
```
nano /srv/homelab/control/caddy/Caddyfile
```

```
{
	auto_https off
}

openwebui.lab {
	reverse_proxy openwebui:8080
}

search.lab {
	reverse_proxy searxng:8080 {
		header_up X-Forwarded-For {remote_host}
		header_up X-Real-IP {remote_host}
		header_up X-Forwarded-Proto {scheme}
		header_up X-Forwarded-Host {host}
	}
}
```
#### docker-compose.yml
```
nano /srv/homelab/control/caddy/docker-compose.yml
```

```
services:
  caddy:
    image: caddy:2
    container_name: caddy
    restart: unless-stopped
    ports:
      - "80:80"
    volumes:
      - /srv/homelab/control/caddy/Caddyfile:/etc/caddy/Caddyfile:ro
      - /srv/homelab/control/caddy/data:/data
      - /srv/homelab/control/caddy/config:/config
    networks:
      - ai-internal

networks:
  ai-internal:
    external: true
```
#### Start Caddy
```
cd /srv/homelab/control/caddy
docker compose up -d
```
#### Test Caddy
```
http://openwebui.lab
http://search.lab
```
