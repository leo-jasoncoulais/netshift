# 05 — Règles de pare-feu (iptables)

Toutes les règles sont gérées via **iptables** (nf_tables backend). Aucun secret, IP réelle ou nom de domaine réel n'est reproduit ci-dessous.

## Netguard (VPS)

### Politique par défaut

| Chaîne | Politique |
|---|---|
| INPUT | DROP |
| FORWARD | DROP |
| OUTPUT | ACCEPT |

### Règles INPUT (autorisations explicites)

| Source | Protocole/Port | Action | Commentaire |
|---|---|---|---|
| Interface loopback | tout | ACCEPT | Loopback |
| Interface WireGuard | tout | ACCEPT | Tunnel WireGuard — trafic interne de confiance |
| Interface publique | ICMP | ACCEPT | Diagnostic réseau |
| Interface publique | TCP/80, TCP/443 | ACCEPT | HTTP/HTTPS (Caddy) |
| Interface publique | UDP/51820 | ACCEPT | Port d'écoute WireGuard |
| Interface publique | TCP/25565 | ACCEPT | Serveur Minecraft (exposition directe) |
| Interface WireGuard | UDP/5201 | ACCEPT | `iperf3` — tests de débit ponctuels ⚠️ à fermer hors utilisation |

### Règles FORWARD (routage inter-réseaux)

| Source | Destination | Action | Bornée ? |
|---|---|---|---|
| `172.30.0.0/16` | `172.30.0.0/16` | ACCEPT | ✅ Oui — couvre en une seule règle tous les échanges internes (partiel/full-tunnel) |
| Réseau full-tunnel | Toute destination | ACCEPT | ⚠️ Bornée en source seulement — nécessaire à la sortie internet des peers full-tunnel |
| Toute source | Debbie, port Minecraft (TCP) uniquement | ACCEPT | ✅ Oui — restreinte au seul port Minecraft |

➡️ **Amélioration par rapport à la version précédente** : l'accès externe à Debbie est désormais borné au port Minecraft uniquement, alors qu'il était auparavant inconditionnel (tout protocole/port). Le risque identifié lors du précédent audit — un futur réseau tiers (`172.40.0.0/16`) capable d'atteindre Debbie ou le réseau full-tunnel sans restriction — est résolu pour Debbie, et non applicable côté full-tunnel puisque cette règle ne fait que laisser sortir le trafic des peers concernés (bornée en source), sans ouvrir d'entrée pour un tiers externe.

### NAT

| Règle | Détail |
|---|---|
| DNAT | TCP/25565 → Debbie (redirection Minecraft) |
| MASQUERADE | Sortie via l'interface publique et l'interface WireGuard (permet le full-tunnel des peers concernés) |

## Debbie (serveur principal)

### Politique par défaut

| Chaîne | Politique |
|---|---|
| FORWARD | DROP |

### Protection anti-spoofing (table `raw`)

Chaque conteneur Docker exposé sur une IP de bridge dédiée est protégé par une règle `PREROUTING ... DROP` qui rejette tout paquet à destination de son IP interne **n'entrant pas par son bridge d'origine**. Cela empêche qu'un service accessible sur son IP de bridge Docker soit atteint via un chemin détourné (autre interface), et force tout accès externe à transiter strictement par les règles DNAT associées à l'IP WireGuard de Debbie.

### Redirections NAT (DNAT) — IP WireGuard de Debbie vers conteneurs internes

| Port exposé (Debbie) | Destination interne | Service |
|---|---|---|
| 8082 | Bridge dédié : 80 | Vaultwarden |
| 8084 | Bridge dédié : 3000 | Obsidian (LiveSync) |
| 8085 | Bridge dédié : 80 / 53 | Pi-hole (web + DNS) |
| 8086 | Bridge dédié : 8008 | Matrix (Synapse) |
| 8087 | — | RustyPaste |
| 8088 | Adresse interne atypique, hors plage Docker standard | Speakeasy (cocktails) |
| 8000 / 9443 | Bridge Docker par défaut | Portainer |
| 8089, 8280 | Bridge dédié | *(projet personnel non documenté)* |

### Isolation des bridges Docker

Chaque réseau Docker (bridge dédié) est isolé par défaut (`DROP` sur le trafic ne provenant pas de son propre bridge), avec acceptation explicite du trafic `RELATED,ESTABLISHED` et des flux entrants définis nommément dans `DOCKER-FORWARD`. C'est une isolation standard générée par le moteur Docker (pas une règle manuelle), garantissant qu'un conteneur ne peut pas nativement joindre le réseau d'un autre bridge sans règle DNAT explicite.

## Limites actuelles

- Aucune segmentation entre peers WireGuard : tout peer connecté au VPN (partiel ou full) peut atteindre l'ensemble des ports exposés sur Debbie.
- Aucun IDS/IPS (Suricata, CrowdSec, Fail2ban) n'est déployé sur les nœuds exposés publiquement.

## Bonne pratique constatée

✅ **Aucun port SSH (22) n'est ouvert sur l'interface publique** côté Netguard : l'administration du VPS ne se fait que depuis l'intérieur du réseau (interface WireGuard), ou via la console hors-bande du fournisseur (Hetzner) en dernier recours. Cela réduit significativement la surface d'attaque exposée publiquement, en évitant un vecteur d'attaque classique (brute-force SSH, scan de credentials).
