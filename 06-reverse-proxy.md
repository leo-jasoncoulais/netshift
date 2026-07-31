# 06 — Reverse proxy (Caddy)

Caddy tourne sur **Netguard** (VPS) et constitue l'unique point d'entrée HTTP/HTTPS de l'infrastructure. Les certificats TLS sont obtenus via challenge DNS Cloudflare (`tls { dns cloudflare ... }`), permettant l'émission de certificats même pour les domaines non exposés publiquement.

## Table des domaines

| Domaine                 | Bind        | Backend                         | Accès restreint (VPN) | Notes                                                               |
| ----------------------- | ----------- | ------------------------------- | --------------------- | ------------------------------------------------------------------- |
| `wg.example.com`        | 172.30.0.1  | `localhost:5000`                | ✅                    | Interface `wireguard-ui`                                            |
| `matrix.example.com`    | IP publique | `172.30.0.2:8086`               | ❌                    | Fédération Matrix + `.well-known`                                   |
| `share.example.com`     | IP publique | `172.30.0.2:8087`               | ❌ (download)         | RustyPaste — upload restreint applicativement au VPN                |
| `speakeasy.example.com` | IP publique | `172.30.0.2:8088`               | ❌                    | Basic Auth supplémentaire (bcrypt)                                  |
| `vw.example.com`        | 172.30.0.1  | `172.30.0.2:8082`               | ✅                    | Vaultwarden                                                         |
| `portainer.example.com` | 172.30.0.1  | `172.30.0.2:9443` (TLS interne) | ✅                    | `tls_insecure_skip_verify` car certificat auto-signé côté Portainer |
| `obsidian.example.com`  | 172.30.0.1  | `172.30.0.2:8084`               | ✅                    | LiveSync                                                            |
| `home.example.com`      | 172.30.0.1  | `172.30.0.2:8123`               | ✅                    | Home Assistant                                                      |
| `dns.example.com`       | 172.30.0.1  | `172.30.0.2:8085`               | ✅                    | Interface Pi-hole                                                   |

## Mécanisme de restriction VPN

Pour les domaines bindés sur `172.30.0.1`, Caddy applique une double protection :

1. **Non-routabilité** : l'IP `172.30.0.1` n'étant pas publiquement routable, seuls les peers connectés au VPN peuvent physiquement l'atteindre, même si le DNS public résout ce domaine vers cette IP.
2. **Filtrage applicatif explicite** :

```caddyfile
@blocked not remote_ip 172.30.0.0/24 172.30.10.0/24
handle @blocked {
    respond 403
}
```

Cette règle est une défense en profondeur : elle protège notamment contre un scénario de mauvaise configuration réseau qui rendrait l'IP accessible autrement qu'via le VPN.

## Cas particuliers

- **Minecraft** ne passe pas par Caddy : le protocole n'étant pas HTTP, l'exposition se fait directement via DNAT iptables sur Netguard (voir [05 — Règles de pare-feu](05-firewall-rules.md)).
- **Speakeasy** est protégé par une couche `basicauth` supplémentaire malgré son exposition publique, en plus de son authentification applicative éventuelle.
- **RustyPaste** a un comportement asymétrique : `share.example.com` (bind IP publique) sert uniquement au téléchargement de fichiers déjà publiés ; la publication elle-même nécessite d'être connecté au VPN (contrainte imposée côté application, pas au niveau de ce vhost).
