# Déploiement rootless derrière BunkerWeb

À exécuter en tant que `deploy` (UID 1001), avec son contexte Docker rootless.
Le root des conteneurs correspond à ce compte sur l'hôte. Ne pas remettre
`user: 1001` dans le Compose.

```sh
# Renseigner les variables de example.env dans la stack Komodo.
# Injecter CF_DNS_API_TOKEN comme secret Komodo.
install -d -m 700 data
touch data/acme.json
chmod 600 data/acme.json

docker info --format '{{json .SecurityOptions}}' # doit contenir rootless
test -S /run/user/1001/docker.sock
docker compose config --quiet
docker compose up -d --wait
docker compose logs --tail=100 socket-proxy traefik
```

Le token Cloudflare nécessite `Zone:Read` et `DNS:Edit` uniquement sur les zones
concernées. Les secrets et l'état ACME ne doivent pas être versionnés. Chaque
serveur conserve son propre `data/acme.json` (0600) ; ne pas le partager entre
instances Traefik. Le challenge DNS-01 nécessite des sorties HTTPS et DNS,
mais aucun port entrant accessible à Let's Encrypt.

## BunkerWeb

BunkerWeb termine le HTTPS public sur 443 et rejoint Traefik en **HTTPS sur
8443**. Configurer l'upstream avec le SNI du domaine demandé et la vérification
du certificat activée. Préserver `Host`, `X-Forwarded-Host`,
`X-Forwarded-Proto: https` et la chaîne `X-Forwarded-For`. Activer le passage des
WebSockets et des délais compatibles avec les flux longs côté WAF.

Renseigner `BIND_IP` avec l'interface privée accessible à BunkerWeb, ou
`127.0.0.1` si le WAF y accède depuis l'hôte lui-même. Un conteneur BunkerWeb
ne peut pas joindre la loopback de l'hôte via sa propre loopback.
Limiter l'accès aux ports publiés au WAF sur le réseau/pare-feu de l'hôte.
`PUBLIC_HTTPS_PORT=443` est le port des clients ; `HTTPS_PORT=8443` celui publié
par Docker. Traefik écoute aussi sur 8443 à l'intérieur, sans capacité Linux.

`EXTERNAL_PROXY` doit contenir les adresses sources **vues par Traefik**, séparées
par des virgules. Le port forwarding rootless peut remplacer l'IP source :
vérifier sur le serveur. Ne pas faire confiance à `0.0.0.0/0`. La confiance dans
les en-têtes ne remplace pas la restriction réseau aux seuls accès BunkerWeb.

## API

L'API `https://TRAEFIK_HOST/api/...` utilise ForwardAuth vers l'outpost Authentik
existant. Définir `AUTHENTIK_OUTPOST_URL` sans slash final, avec un certificat
valide si HTTPS ; l'URL doit être joignable depuis Traefik. Le domaine
`TRAEFIK_HOST` doit parvenir à cette instance depuis BunkerWeb.

Créer un Proxy Provider Authentik en mode **Forward auth (single application)**,
External host `https://TRAEFIK_HOST`, et l'associer à une application et à l'outpost.
Lier une policy/un groupe dédié à cette application pour limiter l'accès à l'API.
La configuration route `/outpost.goauthentik.io/` vers cet outpost sans boucle
d'authentification. Un provider domain-level existant est aussi utilisable,
mais ses permissions sont communes aux applications de son domaine.

Ce raccordement suit l'intégration ForwardAuth Authentik : il utilise sa session
SSO, sans ajouter un client OIDC ni un outpost dans cette stack. Si l'outpost est
indisponible, l'API reste inaccessible. Le dashboard reste désactivé.

Aucun port d’administration n’est publié séparément.

## Services

Sur chaque serveur, démarrer Traefik avant les stacks applicatives. Dans chaque
stack, déclarer le réseau existant `traefik` avec `external: true`, puis y rattacher
les services exposés et ajouter les labels suivants (exemple Jellyfin) :

```yaml
labels:
  traefik.enable: "true"
  traefik.docker.network: "traefik"
  traefik.http.routers.jellyfin.rule: "Host(`jellyfin.example.com`)"
  traefik.http.routers.jellyfin.entrypoints: "websecure"
  traefik.http.routers.jellyfin.tls.certresolver: "letsencrypt"
  traefik.http.routers.jellyfin.service: "jellyfin"
  traefik.http.services.jellyfin.loadbalancer.server.port: "8096"
```

Les services peuvent rester en HTTP derrière Traefik. Les WebSockets sont pris
en charge automatiquement. Déclarer Traefik dans les Known Proxies de Jellyfin.
Les middlewares d'administration ne sont pas appliqués aux applications ; chaque
service garde sa propre politique d'authentification. Utiliser des noms de
routers/services uniques dans l'ensemble des stacks d'un même serveur.
Le réseau Docker local ne permet pas la découverte des services d'autres serveurs.

Tecnativa n'autorise que les familles `CONTAINERS`, `EVENTS`, `PING`, `VERSION`, en
lecture. `CONTAINERS` inclut toutefois plus que list/inspect (lectures de logs,
archives et variables d'environnement notamment) : cette version ne fournit pas
une liste d'URL strictement limitée aux seuls appels Traefik. Aucun service
applicatif ne doit rejoindre `traefik-socket-proxy`.

Sources : [mapping rootless Docker](https://docs.docker.com/engine/security/rootless/uid-gid-mapping/),
[Authentik / Traefik](https://docs.goauthentik.io/add-secure-apps/providers/proxy/server_traefik/),
[Cloudflare / lego](https://go-acme.github.io/lego/dns/cloudflare/).
