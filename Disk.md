# PVE — Architecture et plan de stockage

## Objectif

Ce document décrit l'organisation du stockage du serveur Proxmox VE principal.

L'objectif est de séparer les workloads selon leur type d'utilisation :

- **Samsung SATA 250G** : système Proxmox et petits services permanents.
- **SATA SSD 120G** : applications et bases de données.
- **Patriot P310 NVMe 480G** : workloads nécessitant davantage d'I/O et VM lourdes.
- **Barracuda HDD 2T** : stockage volumineux.
- **LiteOn NVMe 128G** : futur disque système du serveur Proxmox Backup Server.

Les différents SSD restent indépendants. Aucun VG ne doit être étendu sur plusieurs disques physiques.

---

# Plan de stockage

```text
PVE — STOCKAGE FINAL
│
├── Samsung SATA 250G ───────────── SYSTEM / SERVICES
│   │
│   ├── PVE root                    ~55–60G
│   ├── ISO / templates             ~20G conceptuels
│   │
│   └── local-lvm (LVM-thin)
│       ├── Nginx + Cloudflared       4G
│       ├── Pi-hole                   4G
│       ├── SearXNG                   6G
│       ├── Keycloak                 12G
│       ├── Discord Bot #1            4G
│       ├── Discord Bot #2            4G
│       ├── Web #1                    4G
│       └── Web #2                    4G
│
├── SATA SSD 120G ───────────────── APPS / DB
│   │
│   └── apps-db (LVM-thin)
│       ├── Nextcloud app          ~12–16G
│       ├── Nextcloud DB           ~10–16G
│       ├── Redis                    ~4G
│       └── Gitea                  ~20–30G
│
├── Patriot P310 NVMe 480G ──────── FAST / HEAVY
│   │
│   └── fast (LVM-thin)
│       ├── Crafty + Minecraft       ~80G
│       ├── Gitea Runner VM         ~100G
│       ├── OBS VM                  ~100G
│       └── IA / futur               reste
│
├── Barracuda HDD 2T ─────────────── BULK
│   │
│   └── vg_bulk
│       ├── nextcloud_data           ~1T
│       ├── general                 ~400G
│       ├── archives                ~200G
│       └── FREE                    ~300G+
│
└── LiteOn NVMe 128G ─────────────── FUTUR PBS
    │
    └── Proxmox Backup Server
        └── OS uniquement
            (datastore backup sur futur disque séparé)
```

---

# 1. Samsung SATA 250G — SYSTEM / SERVICES

Le Samsung SATA 250G est le disque système du serveur PVE.

Il héberge :

- Proxmox VE ;
- `/var/lib/vz` / stockage `local` ;
- les ISO et templates LXC ;
- le thin-pool `local-lvm` ;
- les petits services permanents.

Les services placés ici sont principalement des LXC peu gourmands en stockage et en I/O :

```text
Nginx + Cloudflared     4G
Pi-hole                 4G
SearXNG                 6G
Keycloak               12G
Discord Bot #1          4G
Discord Bot #2          4G
Web #1                  4G
Web #2                  4G
```

Le but est de conserver sur ce disque les services d'infrastructure légers qui doivent généralement être disponibles en permanence.

Nginx/Cloudflared, DNS et authentification restent ainsi indépendants des disques contenant les workloads lourds.

Les volumes LXC utilisent de préférence :

```text
discard
noatime
```

Les conteneurs doivent rester **unprivileged** sauf besoin particulier.

---

# 2. SATA SSD 120G — APPS / DB

Storage Proxmox :

```text
apps-db
```

Type :

```text
LVM-thin
```

Ce SSD est réservé aux applications persistantes et à leurs bases de données.

Répartition prévue :

```text
Nextcloud app        ~12–16G
Nextcloud DB         ~10–16G
Redis                  ~4G
Gitea                ~20–30G
```

Les données utilisateurs volumineuses de Nextcloud ne doivent pas être stockées ici.

Le principe est :

```text
Nextcloud
│
├── application ─────── SSD 120G
├── PostgreSQL/DB ───── SSD 120G
├── Redis ───────────── SSD 120G
│
└── données users ───── HDD 2T
```

Le SSD fournit donc les I/O nécessaires à l'application et à la base de données, tandis que le HDD fournit la capacité.

---

# 3. Patriot P310 NVMe 480G — FAST / HEAVY

Storage Proxmox :

```text
fast
```

Type :

```text
LVM-thin
```

Le P310 est le stockage rapide principal.

Il est réservé aux workloads ayant besoin de bonnes performances disque ou d'une capacité virtuelle importante :

```text
Crafty + Minecraft      ~80G
Gitea Runner VM         ~100G
OBS VM                  ~100G
IA / futur               reste
```

## Minecraft

Crafty et le serveur Minecraft moddé sont placés sur le NVMe afin d'obtenir de bonnes performances sur :

- chargement/génération des chunks ;
- mods ;
- démarrage du serveur ;
- sauvegardes temporaires ;
- opérations impliquant beaucoup de petits fichiers.

Les sauvegardes Minecraft définitives ne doivent cependant pas rester uniquement sur ce NVMe.

Elles devront être envoyées vers le stockage bulk et/ou le futur PBS.

## Gitea Runner

Le Runner est considéré comme un workload temporaire et reconstructible.

Il peut donc utiliser le NVMe sans nécessiter le même niveau de protection que Gitea lui-même.

## OBS / GPU

La future VM OBS sera également hébergée ici.

Elle recevra ultérieurement le passthrough de l'Intel Arc A770 ainsi que du périphérique de capture USB.

Le même stockage pourra accueillir une VM ou un environnement IA utilisant l'Arc A770 lorsque la carte n'est pas attribuée à OBS.

---

# 4. Barracuda HDD 2T — BULK

Le HDD n'est pas utilisé comme LVM-thin.

Organisation :

```text
vg_bulk
│
├── nextcloud_data     ~1T
├── general           ~400G
├── archives          ~200G
│
└── FREE              ~300G+
```

Une partie du VG doit volontairement rester libre.

Cela permet de modifier l'organisation plus tard sans avoir alloué immédiatement toute la capacité physique du disque.

## Nextcloud

`nextcloud_data` contient les fichiers utilisateurs Nextcloud.

L'application Nextcloud et sa DB restent sur SSD.

Le LV du HDD sera monté sur l'hôte puis exposé au conteneur Nextcloud via un mount point LXC.

Les UID/GID et l'ID mapping devront être vérifiés au moment de cette configuration afin que le conteneur unprivileged puisse correctement accéder aux fichiers.

## General

`general` sert au stockage général nécessitant beaucoup de capacité mais peu de performances NVMe.

## Archives

`archives` est destiné aux données froides ou anciennes qui doivent être conservées mais rarement utilisées.

---

# 5. LiteOn NVMe 128G — FUTUR PBS

Le LiteOn 128G n'appartient pas au stockage du PVE principal à terme.

Il sera réutilisé comme disque système de la future machine :

```text
Proxmox Backup Server
```

Le LiteOn étant ancien et déjà fortement utilisé, son rôle sera limité au système PBS.

Les sauvegardes elles-mêmes devront être placées sur un **autre disque dédié au datastore PBS**.

Architecture finale prévue :

```text
PVE principal
     │
     │ backup réseau
     ▼
PBS
├── LiteOn 128G ───── OS PBS
│
└── futur disque ──── datastore backups
```

Le LiteOn ne doit pas être effacé tant que la migration du nouveau PVE n'est pas entièrement validée, puisqu'il peut encore servir de solution de rollback.

---

# Politique de stockage

La règle générale de l'infrastructure est :

```text
Samsung 250G
    ↓
OS + infrastructure + petits services

SATA SSD 120G
    ↓
Applications + bases de données

P310 NVMe 480G
    ↓
Workloads lourds + VM + gros I/O

Barracuda HDD 2T
    ↓
Données volumineuses

LiteOn 128G
    ↓
OS du futur PBS
```

Les disques ne sont pas agrégés entre eux.

Chaque support conserve une fonction claire afin qu'une panne ou une modification de stockage reste facile à diagnostiquer et à gérer.

---

# LVM-thin et allocations

Les tailles attribuées aux CT/VM sur `local-lvm`, `apps-db` et `fast` représentent la **taille maximale des volumes virtuels**.

Avec LVM-thin, l'intégralité de cette capacité n'est pas nécessairement consommée physiquement immédiatement.

Cela permet une certaine quantité d'overprovisioning.

Cependant :

```text
Data% ≈ 100%  → DANGER
Meta% ≈ 100%  → DANGER
```

Les thin-pools doivent donc être surveillés régulièrement.

Commandes utiles :

```bash
lvs
pvs
vgs
```

Pour davantage de détails :

```bash
lvs -a -o +devices
```

---

# Backups

Aucun stockage situé dans le serveur PVE principal ne doit être considéré comme une sauvegarde complète de ce même serveur.

En particulier :

```text
copie sur le HDD 2T
≠
backup indépendant
```

Le futur PBS constituera la véritable destination de sauvegarde indépendante.

Les éléments importants à protéger comprennent notamment :

```text
PVE / configuration
Nginx
Keycloak
Gitea + DB
Nextcloud + DB + données
Minecraft worlds/config
services personnels
```

Pour les bases de données, privilégier les dumps cohérents (`pg_dump`, `mariadb-dump`, etc.) ou des mécanismes de backup adaptés plutôt qu'une simple copie des fichiers d'une DB en fonctionnement.

---

# Résumé

```text
                    PROXMOX VE
                        │
        ┌───────────────┼────────────────┐
        │               │                │
        ▼               ▼                ▼
   SERVICES          APPS/DB          HEAVY
 Samsung 250G       SATA 120G       P310 480G
        │               │                │
        │               │                │
        └───────────────┬┴────────────────┘
                        │
                        ▼
                   BULK DATA
                  Barracuda 2T
                        │
                        │ backups
                        ▼
                  FUTUR PBS
             LiteOn 128G + datastore
```

Cette organisation privilégie la simplicité : chaque disque possède un rôle déterminé, les services légers sont séparés des workloads lourds, les bases de données restent sur SSD et les données volumineuses utilisent le HDD.
