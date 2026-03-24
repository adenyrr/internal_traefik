# Principe

Chaque stack est exposée via un traefik qui expose les services en HTTPS, via la PKI interne (serveur ACME step-ca). Le but *in fine* est d'éviter toute rupture HTTPS, même interne.

Ce repo contient uniquement le traefik déployé sur chaque stack.

# Schéma d'infrastructure

![Schéma d'infrastructure](assets/infra_diagram.png)

# Utilisation :

## Cloner le dépot

## Définir les permissions

# Variables à définir :

TRAEFIK_HOST : URL de contact pour les metrics et l'api traefik
ACME_URL : URL du serveur ACME de la PKI
ACME_EMAIL : mail associé à la PKI

# Arborescence :

traefik/
├── compose.yaml
├── acme.json          # touch acme.json && chmod 600 acme.json
└── certs/
    └── ca.crt         # CA root de la PKI

Récupérer le root-ca.crt du serveur ACME pour ajouter la chaine de certificat :

```sh
docker exec traefik sh -c "wget --no-check-certificate -O /cert/ca.crt https://acme.domain.internal/roots"
```
