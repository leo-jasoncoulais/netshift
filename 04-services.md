# 04 — Services hébergés

Tous les services sont conteneurisés (Docker) et hébergés sur **Debbie** (172.30.0.2), à l'exception du reverse proxy qui réside sur Netguard.

## Liste des services

| Service                                 | Rôle                                                 | Exposition publique                                                              | Domaine                                |
| --------------------------------------- | ---------------------------------------------------- | -------------------------------------------------------------------------------- | -------------------------------------- |
| **Portainer**                           | Gestion des conteneurs Docker                        | ❌ Interne au VPN uniquement                                                     | `portainer.example.com` (accès filtré) |
| **Bot Discord**                         | Bot applicatif personnel                             | ❌ Aucune (connexion sortante vers l'API Discord, aucun port entrant nécessaire) | —                                      |
| **Matrix (Synapse)**                    | Serveur de messagerie fédérée                        | ✅ Public (via Cloudflare proxy)                                                 | `matrix.example.com`                   |
| **Home Assistant**                      | Domotique                                            | ❌ Interne au VPN uniquement                                                     | `home.example.com` (accès filtré)      |
| **Pi-hole**                             | Résolveur DNS interne / blocage publicitaire         | ❌ Interne au VPN uniquement                                                     | `dns.example.com` (accès filtré)       |
| **Vaultwarden**                         | Gestionnaire de mots de passe (Bitwarden-compatible) | ❌ Interne au VPN uniquement                                                     | `vw.example.com` (accès filtré)        |
| **Application cocktails** ("Speakeasy") | Application personnelle (recettes de cocktails)      | ✅ Public (via Cloudflare proxy) + Basic Auth                                    | `speakeasy.example.com`                |
| **Serveur Minecraft**                   | Serveur de jeu pour amis                             | ✅ Public — IP directe du VPS (TCP brut, non proxifiable par Cloudflare)         | `<IP publique VPS>:25565`              |
| **Serveur Obsidian (LiveSync)**         | Synchronisation de notes                             | ❌ Interne au VPN uniquement                                                     | `obsidian.example.com` (accès filtré)  |
| **RustyPaste**                          | Partage de fichiers/pastebin                         | ⚠️ Mixte : upload restreint au VPN, téléchargement public                        | `share.example.com`                    |

## Logique d'exposition

Deux mécanismes d'exposition coexistent, tous deux gérés par **Caddy** sur Netguard :

1. **Bind sur IP publique du VPS** (masquée dans ce document) avec **proxy Cloudflare (nuage orange)** devant : Matrix, RustyPaste (download), Speakeasy. Cloudflare masque l'IP réelle du VPS pour ces services côté visiteur ; elle n'est volontairement pas reproduite ici puisque ce repository est public.
2. **Bind sur IP WireGuard interne** (`172.30.0.1`) : Portainer, Vaultwarden, Obsidian, Home Assistant, Pi-hole, RustyPaste (upload). Le DNS Cloudflare pointe publiquement vers cette IP privée (non proxifiée), donc résolvable par tous mais **injoignable depuis l'extérieur du VPN** (IP non routable sur internet). Une règle Caddy `@blocked` filtre en complément par IP source (`172.30.0.0/24`, `172.30.10.0/24`), en défense en profondeur.

Le **serveur Minecraft** déroge à ce schéma : le protocole (TCP brut) n'étant pas proxifiable par Cloudflare, le port 25565 est ouvert directement sur l'IP publique du VPS, avec DNAT vers Debbie. Un accès via le VPN reste possible mais n'apporte aucun avantage particulier.

## Cas particuliers non documentés

Deux conteneurs supplémentaires tournent sur Debbie, liés à un projet personnel non documenté dans ce repository à la demande de son propriétaire.
