# 04 - Docker : Uptime Kuma et ntfy

## Objectif
Faire tourner **Uptime Kuma** (supervision) et **ntfy** (notifications) dans des conteneurs, **sans exposer leurs ports**, avec des **données persistantes**.

## Machine
Serveur cloud Ubuntu 24.04.

## 1. Installer Docker
```bash
sudo apt update
sudo apt install -y docker.io docker-compose-v2
sudo systemctl enable --now docker
docker --version && docker compose version
```
> Sur Ubuntu, le paquet `docker-compose` est l'ancienne v1 : utiliser `docker-compose-v2` (commande `docker compose`).

## 2. Créer la pile
```bash
sudo mkdir -p /srv/plateforme/data/{caddy,kuma,ntfy}
cd /srv/plateforme
sudo tee compose.yaml > /dev/null <<'YAML'
services:
  caddy:
    image: caddy:2
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - ./data/caddy/data:/data
      - ./data/caddy/config:/config
    depends_on: [kuma, ntfy]

  kuma:
    image: louislam/uptime-kuma:2
    restart: unless-stopped
    volumes:
      - ./data/kuma:/app/data

  ntfy:
    image: binwiederhier/ntfy
    command: serve
    restart: unless-stopped
    environment:
      - NTFY_BASE_URL=https://notif.<DOMAINE>
      - NTFY_BEHIND_PROXY=true
    volumes:
      - ./data/ntfy/var:/var/cache/ntfy
      - ./data/ntfy/etc:/etc/ntfy
YAML
```
> Kuma et ntfy n'ont **pas de section `ports`** : seul Caddy est joignable depuis l'extérieur (voir exercice 6 pour le `Caddyfile`).

```bash
sudo docker compose up -d
sudo docker compose ps
```

## Vérification
```bash
cd /srv/plateforme && sudo docker compose down && sudo docker compose up -d && sleep 15 && sudo docker compose ps
```
Attendu : les trois conteneurs `Up`, ports publiés **uniquement** sur `caddy` (80/443), Kuma `healthy`. La configuration de Kuma est toujours là après le `down` / `up`.

## Pièges
- Ne pas déclarer de volumes nommés en plus des dossiers `./data` : on risque d'écraser les données existantes.
- Les données sont dans `./data` : ce dossier ne doit **jamais** être versionné dans Git.
