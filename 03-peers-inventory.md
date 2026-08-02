# 03 — Inventaire des peers

| Nom | Réseau(x) | Rôle | Matériel | OS | Localisation |
|---|---|---|---|---|---|
| **Netguard** | Hub (`172.30.0.0/24` + `172.30.10.0/24`) | Hub WireGuard + reverse proxy (Caddy) | VPS Hetzner | Debian | Datacenter Hetzner |
| **Debbie** | `172.30.0.0/24` uniquement | Serveur principal — hébergement Docker | Mini-PC (ThinkCentre) | Debian | Domicile |
| **Windaube** | `172.30.0.0/24` + `172.30.10.0/24` | PC de jeu — streaming (Apollo/Kyber) | PC AM4, GPU haut de gamme | Windows | Domicile |
| **Lunalex** | `172.30.0.0/24` + `172.30.10.0/24` | Client Home Assistant + Moonlight | Téléphone | OneUI (Android) | Mobile |
| **Artemis** | `172.30.0.0/24` + `172.30.10.0/24` | Client Home Assistant + Moonlight | Laptop (Vivobook, léger) | Arch Linux | Mobile (déplacements permanents) |
| **GitHub Actions** | `172.30.0.0/24` uniquement | Déploiement CI/CD (SSH vers Debbie) | Runner GitHub-hosted | — | Cloud GitHub |

## Notes

- **Netguard** et **Debbie** sont les deux seuls nœuds fixes de l'infrastructure.
- **Artemis** change en permanence de réseau physique (déplacements), ce qui justifie particulièrement l'usage du `PersistentKeepalive`.
- **GitHub Actions** est un peer purement fonctionnel : sa configuration WireGuard est provisionnée pour la durée du job CI, avec un accès SSH restreint vers Debbie pour les déploiements applicatifs.
- **Windaube ↔ Lunalex/Artemis** : lien fonctionnel de streaming de jeu (Moonlight côté client, Apollo/Kyber côté hôte sur Windaube), qui transite par le tunnel WireGuard partiel entre pairs du réseau VPN (`172.30.0.0/16`).
