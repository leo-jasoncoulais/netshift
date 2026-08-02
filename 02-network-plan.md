# 02 — Plan d'adressage et configuration WireGuard

## Sous-réseaux

L'ensemble du réseau VPN appartient au bloc **`172.30.0.0/16`**. Un bloc **`172.40.0.0/16`** est réservé pour un futur réseau tiers (non utilisé actuellement — voir roadmap de segmentation).

| Sous-réseau | Rôle | NAT | IPv6 |
|---|---|---|---|
| **`172.30.0.0/24`** (partiel) | Uniquement le trafic à destination du réseau interne transite par le tunnel | Aucun (chaque machine directement connectée) | Non utilisé |
| **`172.30.10.0/24`** (full-tunnel) | Tout le trafic internet de la machine transite par le VPS | Masquerade en sortie sur le VPS (interface WireGuard / interface publique) | Non utilisé |

> Un ancien sous-réseau tiers, lié à un usage externe aujourd'hui déprécié, a été remplacé par le réseau full-tunnel actuel. Non documenté davantage ici.

## Implémentation WireGuard

| Paramètre | Valeur | Commentaire |
|---|---|---|
| Implémentation | `wireguard-ui` (image fournie par Hetzner) | Interface de gestion communautaire par-dessus le module noyau WireGuard |
| MTU | **1420** | ✅ Ajusté depuis la valeur par défaut (1400) — conforme à la recommandation ci-dessous |
| PersistentKeepalive | 15s | Adapté aux peers derrière NAT (mobile, domicile) |
| Pré-shared key (PSK) | Activée sur tous les peers | Renforce la résistance post-quantique de la couche d'échange de clés |
| Rotation des clés | Aucune | Clés statiques depuis la création des peers |
| Sauvegarde des clés privées | Aucune | Perte de fichier = perte définitive de la clé, nécessite un reprovisionnement |

### MTU — ajustement déjà appliqué

Le MTU est passé de 1400 (valeur par défaut prudente mais sous-optimale) à **1420**. Pour un chemin réseau IPv4 simple (domicile → VPS Hetzner, pas de double encapsulation), l'overhead WireGuard est de 60 octets, ce qui autorise théoriquement un MTU de 1420 sans fragmentation si le chemin supporte du 1500 plein (cas généralement vérifié chez Hetzner).

Procédure de calcul empirique :
```bash
ping -M do -s 1372 <IP_publique_VPS>
```
Augmenter/diminuer la taille jusqu'au seuil de fragmentation, puis `MTU_WireGuard = seuil + 28`.

Un MTU plus élevé réduit l'overhead par paquet, ce qui est pertinent pour du streaming à faible latence (Moonlight/Apollo/Kyber) où chaque fragmentation supplémentaire ajoute de la latence de traitement.

## Répartition des configurations par peer

| Peer | `172.30.0.0/24` (partiel) | `172.30.10.0/24` (full-tunnel) |
|---|:---:|:---:|
| Netguard (VPS) | — (hub) | — (hub) |
| Debbie | ✅ | ❌ |
| Windaube | ✅ | ✅ |
| Lunalex | ✅ | ✅ |
| Artemis | ✅ | ✅ |
| GitHub Actions | ✅ | ❌ |

## Filtrage réseau (Netguard — `FORWARD`)

Ruleset en vigueur (mis à jour) :

| Règle | Portée réelle |
|---|---|
| `172.30.0.0/16` → `172.30.0.0/16` | ✅ Bornée — couvre en un seul tenant les échanges internes (partiel ↔ partiel, partiel ↔ full-tunnel, full-tunnel ↔ full-tunnel) |
| `172.30.10.0/24` (full-tunnel) → toute destination | ⚠️ Bornée en source uniquement — nécessaire pour laisser les peers full-tunnel router leur trafic internet général vers l'extérieur |
| Toute source → Debbie, port Minecraft (TCP) uniquement | ✅ Bornée précisément au port Minecraft — remplace l'ancienne règle qui acceptait tout trafic vers Debbie sans restriction de port |

**Amélioration notable par rapport à l'audit précédent** : l'accès à Debbie depuis l'extérieur de `172.30.0.0/16` est désormais restreint au seul port Minecraft, au lieu d'une règle inconditionnelle qui acceptait tout protocole/port. Le risque de contournement d'une future segmentation (`172.40.0.0/16`) via Debbie est donc levé.

La seule règle encore "ouverte" (`172.30.10.0/24` → toute destination) est bornée côté **source** : elle ne permet qu'aux peers full-tunnel d'émettre vers n'importe quelle destination (comportement attendu d'un full-tunnel), mais ne permet pas à une source externe non listée d'entrer. Elle ne pose donc pas de risque symétrique pour une future segmentation, tant qu'aucun tiers n'est lui-même placé dans ce sous-réseau.

En dehors de ce point, l'ensemble du bloc `172.30.0.0/16` constitue un **cercle de confiance unique** : aucune microsegmentation n'est appliquée entre peers internes.
