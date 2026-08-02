# 04 — Services hébergés & reverse proxy

Tous les services sont conteneurisés (Docker) et hébergés sur **Debbie**, à l'exception du reverse proxy (**Caddy**) qui réside sur **Netguard** et constitue l'unique point d'entrée HTTP/HTTPS de l'infrastructure. Les certificats TLS sont obtenus via challenge DNS Cloudflare, permettant l'émission de certificats même pour les domaines non exposés publiquement.

## Liste des services

| Service | Rôle | Exposition publique | Domaine (placeholder) |
|---|---|---|---|
| **Portainer** | Gestion des conteneurs Docker | ❌ Interne au VPN uniquement | `portainer.example.com` |
| **Bot Discord** | Bot applicatif personnel | ❌ Aucune (connexion sortante vers l'API Discord, aucun port entrant nécessaire) | — |
| **Matrix (Synapse)** | Serveur de messagerie fédérée | ✅ Public (via Cloudflare proxy) | `matrix.example.com` |
| **Home Assistant** | Domotique | ❌ Interne au VPN uniquement | `home.example.com` |
| **Pi-hole** | Résolveur DNS interne / blocage publicitaire | ❌ Interne au VPN uniquement | `dns.example.com` |
| **Vaultwarden** | Gestionnaire de mots de passe (Bitwarden-compatible) | ❌ Interne au VPN uniquement | `vw.example.com` |
| **Application cocktails** ("Speakeasy") | Application personnelle (recettes de cocktails) | ✅ Public (via Cloudflare proxy) + Basic Auth | `speakeasy.example.com` |
| **Serveur Minecraft** | Serveur de jeu pour amis | ✅ Public — IP directe du VPS (TCP brut, non proxifiable par Cloudflare) | `<IP publique VPS>:25565` |
| **Serveur Obsidian (LiveSync)** | Synchronisation de notes | ❌ Interne au VPN uniquement | `obsidian.example.com` |
| **RustyPaste** | Partage de fichiers/pastebin | ⚠️ Mixte : upload restreint au VPN, téléchargement public | `share.example.com` |
| **`wireguard-ui`** | Interface de gestion WireGuard | ❌ Interne au VPN uniquement | `wg.example.com` |

## Mécanisme d'exposition via Caddy

Deux mécanismes d'exposition coexistent, tous deux gérés par Caddy sur Netguard :

1. **Bind sur l'IP publique du VPS**, avec **proxy Cloudflare (nuage orange)** devant : Matrix, RustyPaste (download), Speakeasy. Cloudflare masque l'IP réelle du VPS pour ces services côté visiteur.
2. **Bind sur l'IP WireGuard interne de Netguard** : Portainer, Vaultwarden, Obsidian, Home Assistant, Pi-hole, RustyPaste (upload), `wireguard-ui`. Le DNS public résout ces domaines vers cette IP privée (non proxifiée), donc résolvable par tous mais **injoignable depuis l'extérieur du VPN** (IP non routable sur internet). Une règle Caddy `@blocked` filtre en complément par IP source (réseau VPN interne), en défense en profondeur :
   ```caddyfile
   @blocked not remote_ip 172.30.0.0/16
   handle @blocked {
       respond 403
   }
   ```
   Cette règle est une défense en profondeur, complémentaire à la non-routabilité de l'IP.

## Cas particuliers

- **Minecraft** ne passe pas par Caddy : le protocole n'étant pas HTTP, l'exposition se fait directement via DNAT iptables sur Netguard (voir [05 — Règles de pare-feu](05-firewall-rules.md)). Un accès via le VPN reste possible mais n'apporte aucun avantage particulier — Minecraft est destiné à des amis externes au réseau.
- **Bot Discord** : fonctionnement "client" — le bot se connecte à l'API Discord, aucune exposition entrante n'est nécessaire.
- **Speakeasy** est protégé par une couche `basicauth` supplémentaire malgré son exposition publique, en plus de son authentification applicative éventuelle.
- **RustyPaste** a un comportement asymétrique : le téléchargement de fichiers déjà publiés est public ; la publication elle-même nécessite d'être connecté au VPN (contrainte imposée côté application, pas au niveau du vhost Caddy).

