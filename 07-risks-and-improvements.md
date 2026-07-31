# 07 — Risques et pistes d'amélioration

Cette section recense les compromis assumés de l'infrastructure actuelle et les évolutions envisageables, sans jugement de valeur : il s'agit d'une infrastructure personnelle, où simplicité et coût priment sur la haute disponibilité.

## Risques identifiés

| Risque | Impact | Statut |
|---|---|---|
| **VPS = point de défaillance unique** | Si Netguard tombe, l'ensemble des services internes et externes devient injoignable, et les peers en full-tunnel (Windaube, Lunalex, Artemis) perdent tout accès internet | Assumé — pas de redondance prévue |
| **Pas de rotation ni de sauvegarde des clés WireGuard** | La perte d'un fichier de config impose un reprovisionnement manuel du peer concerné | Assumé — mitigation possible via Vaultwarden (déjà en place) |
| **Aucun monitoring des tunnels** | Une panne de tunnel (handshake expiré) peut passer inaperçue plusieurs heures/jours | À améliorer |
| **Aucune centralisation des logs** | Diagnostic post-incident difficile, absence de visibilité sur les tentatives d'intrusion | À améliorer |
| **Aucun IDS/IPS sur les nœuds exposés** | Pas de détection automatique de scan ou de brute-force sur les ports publics (80/443/25565) | À améliorer |
| **Aucune sauvegarde des volumes Docker** | Perte de données définitive en cas de panne disque sur Debbie (Vaultwarden, Matrix, Home Assistant, etc.) | En attente d'un NAS dédié |
| **Fuite d'information via DNS public** | Les enregistrements DNS pointant vers `172.30.0.1` révèlent publiquement le plan d'adressage interne | Mineur — IP non routable, risque limité |
| **Port `iperf3` ouvert en permanence** | Surface d'attaque légèrement augmentée pour un usage ponctuel | Facile à corriger |
| ~~**Règles `FORWARD` inconditionnelles vers/depuis Debbie et `172.30.10.0/24`**~~ | ~~Un futur réseau tiers n'aurait pas été isolé sans réécriture~~ | ✅ **Corrigé** — l'accès externe à Debbie est désormais borné au port `25565/tcp` (Minecraft) uniquement |

## Pistes d'amélioration priorisées

1. **Court terme, faible effort**
   - Fermer le port `iperf3` (UDP 5201) en dehors des sessions de test.
   - Exporter et chiffrer une sauvegarde des clés/configs WireGuard dans Vaultwarden (déjà disponible).
   - Ajuster le MTU à 1420 après validation empirique (cf. [02 — Plan d'adressage](02-network-plan.md)).

2. **Moyen terme**
   - Déployer un monitoring léger des tunnels WireGuard (ex. exporter Prometheus dédié, ou simple script `wg show` + Healthchecks.io/UptimeRobot pour une alerte de handshake expiré).
   - Mettre en place CrowdSec ou Fail2ban sur Netguard pour les services exposés publiquement (Matrix, Speakeasy, RustyPaste, Minecraft).
   - Centraliser les logs Caddy et iptables (Loki ou solution légère équivalente).

3. **Long terme**
   - Acquisition d'un NAS pour sauvegarde régulière des volumes Docker (Restic/Borg).
   - Étudier une solution de failover pour le hub WireGuard (VPS secondaire en veille, DNS à bascule rapide) si la disponibilité devient critique.
   - Formaliser la segmentation prévue pour un futur réseau tiers (`172.40.0.0/24`), avec règles `FORWARD` explicites d'isolation vis-à-vis de `172.30.0.0/16`.
   - Introduire une gestion des configurations WireGuard via Infrastructure as Code (Ansible) pour fiabiliser le provisioning et faciliter la rotation des clés.
