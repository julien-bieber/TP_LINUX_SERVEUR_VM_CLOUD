# TP Linux serveur : de la VM au cloud

Procédure pas à pas pour reproduire le TP « Linux serveur : de la VM au cloud » (CESI).
Chaque fichier correspond à un exercice et suit le même plan : **objectif, machine, étapes, vérification, pièges**.

> Toutes les valeurs propres à un environnement sont remplacées par des variables génériques.
> Remplacez-les par les vôtres avant de lancer les commandes.

## Variables utilisées

| Variable | Exemple | Signification |
|---|---|---|
| `<IP_VM>` | `172.x.x.x` | IP (DHCP) de la VM Debian locale, carte `eth0` |
| `<IP_PRIVEE>` | `192.168.56.10` | IP statique de la VM sur le réseau privé, carte `eth1` |
| `<IP_SERVEUR>` | `51.x.x.x` | IP publique du serveur cloud |
| `<DOMAINE>` | `mondomaine.fr` | Domaine pointant vers le serveur cloud |
| `<UTILISATEUR>` | `alice` | Utilisateur d'administration (sudo) |
| `<SUJET_NTFY>` | `alertes-xxxxxxxx` | Nom de sujet ntfy difficile à deviner |
| `<COMPTE_GITHUB>` | `mon-compte` | Compte GitHub qui héberge le dépôt de la plateforme |

## Machines

| Machine | Rôle | Exercices |
|---|---|---|
| VM locale Debian 13 (Hyper-V) | Bases Linux, réseau, GLPI en LAMP « à la main » | 1, 2, 3 |
| Serveur cloud Ubuntu 24.04 | Docker, sécurité, HTTPS, sauvegardes | 4 à 8 |
| Seconde VM cloud vierge | Test de reconstruction par Ansible | 8 |
| Poste client Windows | Accès SSH, tests de recette | 1, 5, 9 |

## Sommaire

1. [Serveur Linux sécurisé](<01 - Serveur Linux sécurisé.md>)
2. [Réseau couche par couche](<02 - Réseau couche par couche.md>)
3. [GLPI en LAMP](<03 - GLPI en LAMP.md>)
4. [Docker : Uptime Kuma et ntfy](<04 - Docker Kuma et ntfy.md>)
5. [Qui frappe à la porte ?](<05 - Qui frappe à la porte.md>)
6. [HTTPS avec Caddy](<06 - HTTPS avec Caddy.md>)
7. [Sauvegardes restic](<07 - Sauvegardes restic.md>)
8. [Git et Ansible](<08 - Git et Ansible.md>)
9. [Recette finale](<09 - Recette finale.md>)

> Les captures sont dans [`assets/`](assets/). Les IP, domaines, noms d'hôtes et sujets ntfy y sont masqués.

## Arborescence finale sur le serveur cloud

```
/srv/plateforme/
├── compose.yaml        # Caddy + Uptime Kuma + ntfy
├── Caddyfile           # reverse proxy et HTTPS
├── nftables.conf       # pare-feu (copié par Ansible)
├── data/               # données persistantes (non versionnées)
│   ├── caddy/
│   ├── kuma/
│   └── ntfy/
└── ansible/
    ├── ansible.cfg
    ├── inventory.ini
    └── site.yml
```
