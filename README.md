# Stremio Addon Proxy

Un addon-proxy Stremio auto-hébergé. Tu ajoutes tes addons dans son interface web, il te
génère pour chacun un lien « proxifié » que tu colles dans Stremio. Résultat : les flux
vidéo **HTTP/HTTPS** ne sont plus téléchargés par ton appareil, mais **par le serveur**
(donc avec l'IP du serveur — ou celle de WARP/VPN si tu l'actives).

## Comment ça marche

```
Stremio  ──►  Stremio Addon Proxy (ton serveur)  ──►  addon d'origine
                     │
                     └──►  télécharge la vidéo (via IP serveur ou WARP) et la renvoie à Stremio
```

1. Le proxy expose, pour chaque addon, un manifest à l'adresse `/{id}/manifest.json`.
2. Quand Stremio demande les streams, le proxy interroge l'addon d'origine et **réécrit
   les URLs vidéo** pour qu'elles passent par sa route `/play`.
3. `/play` télécharge la vidéo depuis la source (avec les bons headers et le support du
   *Range* pour l'avance/recul) et la renvoie en streaming.

**Limite importante :** seuls les flux **directs HTTP/HTTPS** sont proxifiables. Les flux
**torrent** (`infoHash`, ex. Torrentio) passent par le moteur P2P intégré de Stremio et
ne peuvent pas être routés ainsi — ils sont laissés tels quels.

## Lancer en local (test)

```bash
npm install
npm start
# Interface : http://localhost:7000
```

## Déployer sur ton VPS (Docker)

```bash
docker compose up -d --build
```

L'interface est sur `http://IP_DU_VPS:7000`. La config (addons + réglages) est persistée
dans `./data/config.json`.

### HTTPS (recommandé pour Stremio Web / mobile)

Stremio Web et l'app mobile exigent souvent du HTTPS. Mets le proxy derrière un
reverse-proxy (Caddy, Nginx Proxy Manager, Traefik…) avec un certificat, puis renseigne
l'URL publique — soit via la variable d'env `PUBLIC_URL`, soit dans le champ dédié de
l'interface :

```yaml
environment:
  - PUBLIC_URL=https://proxy.mondomaine.com
```

Exemple Caddy minimal :

```
proxy.mondomaine.com {
    reverse_proxy localhost:7000
}
```

## Sortie réseau : IP serveur ou WARP/VPN

Dans l'onglet **Sortie réseau** de l'interface :

- **Direct** — les vidéos sortent avec l'IP du serveur (par défaut).
- **SOCKS5** — pour router via Cloudflare WARP ou un VPN exposant un SOCKS5.
  Ex. d'URL : `socks5://127.0.0.1:40000`.
- **Proxy HTTP** — ex. `http://user:pass@host:port`.

Le bouton **« Tester l'IP de sortie »** interroge un service d'écho et t'affiche l'IP
publique réellement utilisée : pratique pour vérifier que WARP est bien pris en compte.

### Mettre WARP en SOCKS5

Deux options courantes sur un VPS Linux :

- **`warp-svc` + `warp-cli`** en mode proxy :
  ```bash
  warp-cli set-mode proxy      # expose un SOCKS5 local (par défaut sur 127.0.0.1:40000)
  warp-cli connect
  ```
  Puis dans l'interface : mode **SOCKS5**, URL `socks5://127.0.0.1:40000`.

  ⚠️ Si le proxy tourne dans Docker, `127.0.0.1` désigne le conteneur, pas l'hôte.
  Utilise `network_mode: host` (Linux) ou l'adresse de la passerelle Docker
  (`host.docker.internal` selon la config) pour joindre le WARP de l'hôte.

- **wgcf + wireguard** si tu préfères monter WARP en interface réseau : dans ce cas tu
  peux même laisser le proxy en **Direct**, tout le trafic du VPS sortant déjà par WARP.

### Stack tout-en-un : proxy + WARP en Docker (recommandé pour tester)

Le fichier [`docker-compose.warp.yml`](docker-compose.warp.yml) démarre **le proxy ET un
conteneur WARP** exposant un SOCKS5. Le proxy y sort automatiquement (variables
`UPSTREAM_MODE=socks` / `UPSTREAM_URL=socks5://warp:1080`).

```bash
docker compose -f docker-compose.warp.yml up -d --build
```

Vérifier que la sortie passe bien par Cloudflare :

```bash
# IP de sortie vue via le proxy (doit être une IP Cloudflare, pas celle du VPS)
curl -s -X POST http://localhost:7000/api/test-upstream \
  -H "Content-Type: application/json" \
  -d '{"upstream":{"mode":"socks","url":"socks5://warp:1080"}}'

# Contrôle direct du conteneur WARP (doit afficher warp=on)
docker exec warp curl -s --socks5 localhost:1080 https://cloudflare.com/cdn-cgi/trace | grep warp
```

Ou plus simplement : bouton **« Tester l'IP de sortie »** dans l'interface.

> **Note :** `UPSTREAM_MODE`/`UPSTREAM_URL` ne s'appliquent qu'au **premier** démarrage
> (quand `data/config.json` n'existe pas encore). Ensuite, c'est l'interface qui fait foi.
> Pour forcer de nouveau WARP : change-le dans l'UI, ou supprime `data/config.json`.
>
> Le service WARP a besoin de `/dev/net/tun` et de la capability `NET_ADMIN` — OK sur un
> VPS Linux et sur Docker Desktop (backend WSL2).

## Image Docker prête à l'emploi (GitHub Actions)

Un workflow ([`.github/workflows/docker.yml`](.github/workflows/docker.yml)) construit et
publie une image **multi-arch (amd64 + arm64)** sur le GitHub Container Registry à chaque
push sur `main` (et sur les tags `v*`). Image :

```
ghcr.io/guiro28/stremio-addon-proxy:latest
```

Pour l'utiliser au lieu de builder localement, décommente la ligne `image:` dans le
compose (et retire `build: .`). Le dépôt étant **privé**, l'image l'est aussi : soit tu
rends le *package* public sur GitHub, soit tu t'authentifies sur le VPS avant de tirer :

```bash
echo $GHCR_TOKEN | docker login ghcr.io -u guiro28 --password-stdin
```

(`GHCR_TOKEN` = un *Personal Access Token* GitHub avec le scope `read:packages`.)

## Authentification de l'interface

L'UI et l'API de gestion sont protégées par une **page de login** (session par cookie
signé HMAC, sans dépendance). Définis simplement les variables d'env `AUTH_USER` et
`AUTH_PASS` (voir `docker-compose.yml`). Si elles sont vides, l'interface est ouverte
(un avertissement s'affiche au démarrage).

- Accès à `/` sans session → redirection vers `/login`.
- Après connexion, un cookie `sp_session` (HttpOnly, SameSite=Lax, 7 jours) est posé.
- Bouton **Déconnexion** en haut de l'interface.
- `AUTH_SECRET` (optionnel) : fixe le secret de signature pour que les sessions survivent
  à un changement de mot de passe. Par défaut il est dérivé des identifiants.

Seules `/` et `/api/*` sont protégées : les routes que Stremio consomme
(`/{id}/manifest.json`, `/{id}/*`, `/play`) **restent publiques** — c'est nécessaire,
sinon le lecteur Stremio recevrait un 401 et ne pourrait pas lire les flux.

> Astuce : tu peux aussi ajouter une *Access List* côté Nginx Proxy Manager, mais l'auth
> applicative ci-dessus suffit et voyage avec le conteneur.

## Endpoints (pour info)

| Route | Rôle |
|-------|------|
| `GET /` | Interface web |
| `GET /api/state` | Réglages + liste des addons |
| `POST /api/addons` | Ajoute un addon (`{ manifestUrl }`) |
| `DELETE /api/addons/:id` | Supprime un addon |
| `POST /api/settings` | Change la sortie réseau |
| `POST /api/test-upstream` | Renvoie l'IP publique de sortie |
| `GET /:id/manifest.json` | Manifest proxifié |
| `GET /:id/*` | Ressources proxifiées (streams réécrits) |
| `GET /play?t=…` | Relais vidéo |

## Variables d'environnement

| Variable | Défaut | Rôle |
|----------|--------|------|
| `PORT` | `7000` | Port d'écoute |
| `DATA_DIR` | `./data` | Dossier de persistance |
| `PUBLIC_URL` | — | URL publique (fallback si non défini dans l'interface) |
| `AUTH_USER` | — | Identifiant Basic Auth de l'interface (vide = pas d'auth) |
| `AUTH_PASS` | — | Mot de passe Basic Auth de l'interface |
