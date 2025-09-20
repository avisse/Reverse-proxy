# Projet — Reverse Proxy NGINX pour GLPI & Rocket.Chat (Docker)

## 1. Contexte & objectifs
Point d’entrée unique sécurisé vers GLPI et Rocket.Chat via un reverse proxy **NGINX**.
- URLs homogènes (sous-domaines recommandés).
- Terminaison **TLS** au proxy.
- Les backends ne sont **pas exposés** sur l’hôte.
- Possibilité d’ajouter filtrage, authent, logging.

## 2. Architecture
```
Internet ──> [NGINX reverse-proxy :80/443] ──> [GLPI:80] [Rocket.Chat:3000]
              (réseau Docker partagé: glpinet)
```
Seul NGINX publie des ports vers l’hôte. GLPI & Rocket.Chat ne sont visibles que sur le réseau Docker **glpinet**.

## 3. Pré-requis
- Docker + Docker Compose
- Réseau partagé (une fois) : `docker network create glpinet`
- GLPI & Rocket.Chat déjà déployés (ou à déployer) et **rattachés** au réseau `glpinet`
  - GLPI → MariaDB (stack séparé)
  - Rocket.Chat → MongoDB (stack séparé)

## 4. Installation
1) Copier les variables :
```
cp .env.example .env
```
2) Éditer `.env` (hôtes & upstreams) :
```
GLPI_HOST=glpi.example.com
CHAT_HOST=chat.example.com
GLPI_UPSTREAM=http://glpi:80
CHAT_UPSTREAM=http://rocketchat:3000
```
3) Démarrer le proxy :
```
docker compose up -d
```

## 5. Routage (sous-domaines recommandé)
- Fichiers : `nginx/conf.d/10-glpi-subdomain.conf`, `nginx/conf.d/20-rocketchat-subdomain.conf`
- Gèrent les en-têtes `X-Forwarded-*` et WebSocket pour Rocket.Chat.

> Option “sous-chemins” (exemple) : `nginx/conf.d/90-paths-example.conf`  
> ⚠️ nécessite d’ajuster la baseURL côté applis (GLPI / `ROOT_URL` Rocket.Chat).

## 6. HTTPS (certificats)
- Monter vos certs dans `./certs` et **décommenter** les blocs HTTPS dans les vhosts.
- En prod, privilégier **Let’s Encrypt** (certbot ou Traefik/Caddy).

## 7. Validation
- DNS/hosts vers l’IP du proxy.
- `curl -I http://glpi.example.com` (HTTP→301/HTTPS si activé).
- WebSocket : `curl -I -H "Upgrade: websocket" -H "Connection: upgrade" http://chat.example.com`
- Réseau Docker : `docker network inspect glpinet`
- Logs : `./logs/access.log`, `./logs/error.log`

## 8. Dépannage
- **502 Bad Gateway** : service/port/nom Docker incorrect, service non sur `glpinet`.
- **Assets manquants (mode chemin)** : baseURL non ajustée → préférer sous-domaines.
- **WS Rocket.Chat** : `Upgrade/Connection` + HTTP/1.1 obligatoires (déjà dans la conf).

## 9. Exploitation
```
docker compose logs -f reverse-proxy
docker exec -it reverse-proxy nginx -t
docker exec -it reverse-proxy nginx -s reload
docker compose pull && docker compose up -d
```

## 10. Contenu
- `.env.example` — variables hôtes/upstreams/chemins certs
- `docker-compose.yml` — service reverse proxy
- `nginx/nginx.conf` — conf principale
- `nginx/conf.d/10-glpi-subdomain.conf` — vhost GLPI
- `nginx/conf.d/20-rocketchat-subdomain.conf` — vhost Rocket.Chat (WebSocket)
- `nginx/conf.d/90-paths-example.conf` — routage par chemins (exemple)
- `logs/` — logs NGINX
- `certs/` — certificats si gérés localement
- `.gitignore` — ignore logs, certs, `.env`
