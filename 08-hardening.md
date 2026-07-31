# 08 — Durcissement (hardening)

Ce document recense des mesures de durcissement identifiées mais non toutes mises en œuvre, en complément des risques déjà listés dans [07 — Risques et pistes d'amélioration](07-risks-and-improvements.md).

## Bonnes pratiques déjà en place

| Mesure | Détail |
|---|---|
| **SSH non exposé publiquement** | Aucun port 22 ouvert sur `eth0` de Netguard — administration uniquement via `wg0` ou console hors-bande du fournisseur |
| **Mises à jour manuelles régulières** | Système et conteneurs mis à jour manuellement chaque semaine sur Debbie et Netguard |
| **PSK WireGuard** | Pré-shared key activée sur tous les peers, en complément de l'échange de clés Curve25519 |
| **Isolation Docker par bridge** | Chaque service dispose de son propre réseau Docker, avec anti-spoofing en table `raw` |

## Pistes de durcissement non mises en œuvre

### Restriction du peer GitHub Actions

Le peer CI/CD est le seul non-humain et automatisé de l'infrastructure — un secret GitHub compromis ou une dépendance vérolée dans le pipeline représente un vecteur d'attaque distinct des autres peers. Deux mesures complémentaires, indépendantes l'une de l'autre :

1. **Côté WireGuard** : restreindre l'`AllowedIPs` de ce peer à `172.30.0.2/32` (Debbie uniquement) plutôt qu'à l'ensemble de `172.30.0.0/16`, afin qu'il ne puisse physiquement joindre que sa cible légitime.
2. **Côté SSH (Debbie)** : contraindre la clé utilisée par ce peer via une entrée `authorized_keys` restrictive :
   ```
   command="/usr/local/bin/deploy.sh",no-port-forwarding,no-X11-forwarding,no-agent-forwarding,no-pty ssh-ed25519 AAAA... github-actions@ci
   ```
   Cela force l'exécution d'un script de déploiement précis quelle que soit la commande demandée par le client, et désactive les usages détournés du tunnel SSH (rebond réseau, forwarding de port, etc.).

**Statut** : non mis en place — un utilisateur Linux dédié à droits restreints (plutôt que la restriction `command=` sur la clé) est prévu pour la partie SSH ; la restriction d'`AllowedIPs` reste à faire côté WireGuard.

### En-têtes de sécurité HTTP (Caddy)

Absence actuelle d'en-tête `Strict-Transport-Security` (HSTS) sur les domaines exposés. Cet en-tête force le navigateur à toujours utiliser HTTPS pour le domaine concerné, y compris lors de la toute première visite, réduisant le risque d'interception lors d'un downgrade HTTP→HTTPS (typiquement sur un réseau non fiable).

Exemple d'ajout global via un `snippet` Caddy réutilisable sur l'ensemble des vhosts :
```caddyfile
(security_headers) {
    header {
        Strict-Transport-Security "max-age=31536000; includeSubDomains"
    }
}
```

**Statut** : ✅ **Mis en place**.

### DNSSEC et enregistrement CAA

Deux mécanismes indépendants renforçant la couche DNS, actuellement non vérifiée cryptographiquement :

- **DNSSEC** : signe cryptographiquement la chaîne de résolution DNS depuis la racine jusqu'au domaine, empêchant un empoisonnement de cache DNS qui redirigerait un sous-domaine vers une IP arbitraire avant même que Cloudflare ou Caddy n'interviennent. Activable en un clic côté Cloudflare (`DNS` → `Settings` → `DNSSEC`), qui génère un enregistrement `DS`. ⚠️ Ce `DS` doit ensuite être déclaré **chez le registrar**, pas dans la zone Cloudflare : la délégation du domaine (quels DS/NS font autorité) est une information de la zone parente (registre `.xyz`), distincte du contenu de zone hébergé par Cloudflare. Chercher une section dédiée "DNSSEC" ou "DS Records" dans l'interface du registrar, séparée de la gestion DNS classique.
- **CAA** (`Certification Authority Authorization`) : restreint quelles autorités de certification sont autorisées à émettre un certificat pour le domaine, limitant le risque de mésémission par une CA tierce. Exemple :
  ```
  example.com.  CAA  0 issue "letsencrypt.org"
  ```

**Statut** : non mis en place.

### Portainer — compte à privilèges limités

Portainer dispose d'un accès complet au socket Docker (équivalent root sur l'hôte Debbie). Bien que déjà restreint au VPN, il est recommandé d'utiliser un compte Portainer dédié à droits limités pour les usages courants, en réservant le compte administrateur aux opérations exceptionnelles.

**Statut** : noté, action laissée à l'appréciation du propriétaire de l'infrastructure.

### Sauvegarde de Vaultwarden

Vaultwarden centralise les secrets de l'infrastructure (et pourrait, à terme, héberger une sauvegarde chiffrée des clés WireGuard). Une perte du disque de Debbie sans sauvegarde entraînerait la perte simultanée de tous les secrets et des clés qu'il est censé protéger. Une exportation chiffrée hors-site, même manuelle et peu fréquente, est prioritaire par rapport aux autres sauvegardes en attente du NAS dédié.

**Statut** : en attente, dépend de l'acquisition du NAS (voir [07 — Risques et pistes d'amélioration](07-risks-and-improvements.md)).
