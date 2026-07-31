# 🌐 Infrastructure réseau

Dossier d'Architecture Technique (DAT) documentant mon infrastructure réseau personnelle, bâtie autour d'un hub WireGuard central hébergé sur un VPS Hetzner, desservant un serveur principal et plusieurs machines clientes.

> ⚠️ Ce repository documente l'architecture **sans exposer** de secrets : aucune clé privée, PSK, token ou mot de passe n'y figure. Toutes les IP publiques et éléments sensibles sont partiellement masqués ou génériques lorsque nécessaire.

## Sommaire

| Document | Contenu |
|---|---|
| [01 — Architecture générale](docs/01-architecture.md) | Vue d'ensemble, topologie, schéma réseau |
| [02 — Plan d'adressage](docs/02-network-plan.md) | Sous-réseaux, IP, MTU, WireGuard |
| [03 — Inventaire des peers](docs/03-peers-inventory.md) | Détail des machines connectées |
| [04 — Services hébergés](docs/04-services.md) | Liste des services, exposition, ports |
| [05 — Règles de pare-feu](docs/05-firewall-rules.md) | iptables — VPS & serveur principal |
| [06 — Reverse proxy](docs/06-reverse-proxy.md) | Configuration Caddy, domaines |
| [07 — Risques & pistes d'amélioration](docs/07-risks-and-improvements.md) | SPOF, monitoring, sauvegardes, roadmap |

## En bref

- **Hub central** : VPS Hetzner, point de passage unique (pas de redondance), fait office de routeur WireGuard + reverse proxy.
- **Réseau privé** : `172.30.0.0/16`, découpé en trafic partiel (`172.30.0.0/24`) et full-tunnel (`172.30.10.0/24`).
- **6 peers** : 1 VPS, 1 serveur principal (Docker), 1 PC fixe, 1 laptop, 1 téléphone, 1 peer CI/CD (GitHub Actions).
- **Exposition publique** via Cloudflare (DNS + proxy) et Caddy comme reverse proxy interne.
- **Aucune redondance, aucun monitoring actif** à ce stade — infrastructure personnelle en évolution continue.

## Licence

Documentation publiée à titre de retour d'expérience personnel. Aucune garantie fournie quant à la reproductibilité ou la sécurité de cette architecture pour un usage tiers.
