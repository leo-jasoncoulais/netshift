# 05 — Règles de pare-feu (iptables)

Toutes les règles sont gérées via **iptables** (nf_tables backend). Aucun secret ni IP publique complète n'est reproduit ci-dessous.

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
| `lo` | tout | ACCEPT | Loopback |
| `wg0` | tout | ACCEPT | Tunnel WireGuard — trafic interne de confiance |
| `eth0` | ICMP | ACCEPT | Diagnostic réseau |
| `eth0` | TCP/80, TCP/443 | ACCEPT | HTTP/HTTPS (Caddy) |
| `eth0` | UDP/51820 | ACCEPT | Port d'écoute WireGuard |
| `eth0` | TCP/25565 | ACCEPT | Serveur Minecraft (exposition directe) |
| `wg0` | UDP/5201 | ACCEPT | `iperf3` — tests de débit ponctuels ⚠️ à fermer hors utilisation |

### Règles FORWARD (routage inter-réseaux)

| Source | Destination | Action | Bornée ? |
|---|---|---|---|
| `172.30.0.0/16` | `172.30.0.0/16` | ACCEPT | ✅ Oui — couvre en une seule règle tous les échanges internes (partiel/full-tunnel) |
| `172.30.10.0/24` | **\* (toute destination)** | ACCEPT | ⚠️ Bornée en source seulement — nécessaire à la sortie internet des peers full-tunnel |
| **\* (toute source)** | `172.30.0.2:25565/tcp` (Debbie, Minecraft) | ACCEPT | ✅ Oui — restreinte au seul port Minecraft |

➡️ **Amélioration par rapport à la version précédente** : l'accès externe à Debbie est désormais borné au port `25565/tcp` uniquement, alors qu'il était auparavant inconditionnel (tout protocole/port). Le risque identifié lors du précédent audit — un futur réseau tiers (`172.40.0.0/24`) capable d'atteindre Debbie ou `172.30.10.0/24` sans restriction — est résolu pour Debbie, et non applicable côté `172.30.10.0/24` puisque cette règle ne fait que laisser sortir le trafic des peers full-tunnel (bornée en source), sans ouvrir d'entrée pour un tiers externe.

### NAT

| Règle | Détail |
|---|---|
| DNAT | TCP/25565 → `172.30.0.2` (redirection Minecraft vers Debbie) |
| MASQUERADE | Sortie via `eth0` (interface publique) et `wg0` (permet le full-tunnel des peers `172.30.10.0/24`) |

## Debbie (serveur principal)

### Politique par défaut

| Chaîne | Politique |
|---|---|
| FORWARD | DROP |

### Protection anti-spoofing (table `raw`)

Chaque conteneur Docker exposé sur une IP de bridge dédiée est protégé par une règle `PREROUTING ... DROP` qui rejette tout paquet à destination de son IP interne **n'entrant pas par son bridge d'origine**. Cela empêche qu'un service accessible sur son IP de bridge Docker soit atteint via un chemin détourné (autre interface), et force tout accès externe à transiter strictement par les règles DNAT associées à l'IP WireGuard `172.30.0.2`.

### Redirections NAT (DNAT) — IP WireGuard `172.30.0.2` vers conteneurs internes

| Port exposé (172.30.0.2) | Destination interne | Service |
|---|---|---|
| 8082 | Bridge dédié : 80 | Vaultwarden |
| 8084 | Bridge dédié : 3000 | Obsidian (LiveSync) |
| 8085 | Bridge dédié : 80 / 53 | Pi-hole (web + DNS) |
| 8086 | Bridge dédié : 8008 | Matrix (Synapse) |
| 8087 | — | RustyPaste |
| 8088 | 192.168.0.2:3000 | Speakeasy (cocktails) |
| 8000 / 9443 | `docker0` | Portainer |
| 8089, 8280 | Bridge dédié | *(projet personnel non documenté)* |

### Isolation des bridges Docker

Chaque réseau Docker (`br-xxxxx`) est isolé par défaut (`DROP` sur le trafic ne provenant pas de son propre bridge), avec acceptation explicite du trafic `RELATED,ESTABLISHED` et des flux entrants définis nommément dans `DOCKER-FORWARD`. C'est une isolation standard générée par le moteur Docker (pas une règle manuelle), garantissant qu'un conteneur ne peut pas nativement joindre le réseau d'un autre bridge sans règle DNAT explicite.

## Limites actuelles

- Aucune segmentation entre peers WireGuard : tout peer connecté au VPN (partiel ou full) peut atteindre l'ensemble des ports exposés sur Debbie.
- Aucun IDS/IPS (Suricata, CrowdSec, Fail2ban) n'est déployé sur les nœuds exposés publiquement.

## Bonne pratique constatée

✅ **Aucun port SSH (22) n'est ouvert sur `eth0`** côté Netguard : l'administration du VPS ne se fait que depuis l'intérieur du réseau (`wg0`), ou via la console hors-bande du fournisseur (Hetzner) en dernier recours. Cela réduit significativement la surface d'attaque exposée publiquement, en évitant un vecteur d'attaque classique (brute-force SSH, scan de credentials).
