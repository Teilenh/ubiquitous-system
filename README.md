# Proxmox Homelab

Documentation de mon infrastructure **Proxmox VE**, des services actuellement hébergés et des évolutions prévues.

## Documentation

- [Stockage et disques](./Disk.md)

---

## Services hébergés

| Service | Type | OS | Notes |
|---|---|---|---|
| **SearXNG** | LXC | Alpine Linux | Moteur de recherche / metasearch |
| **Nginx** | LXC | Alpine Linux | Reverse proxy |
| **Cloudflared** | LXC | Alpine Linux | Hébergé dans le même CT que Nginx |
| **Keycloak** | LXC | Alpine Linux | Authentification / gestion des identités |
| **Crafty Controller** | LXC | Debian 13 | Gestion de serveurs Minecraft |
| **Windows VM** | VM | Windows | GPU passthrough |
| **Pi-hole** | LXC | Alpine Linux | DNS / filtrage réseau |
| **Services Python personnels** | LXC | Alpine Linux | Services et outils Python maison |

### Répartition actuelle

```text
Proxmox VE
├── LXC - SearXNG
│   ├── Alpine Linux
│   └── LV : local-lvm
│
├── LXC - Reverse Proxy
│   ├── Alpine Linux
│   │   ├── Nginx
│   │   └── Cloudflared
│   └── LV : apps
│
├── LXC - Keycloak
│   ├── Alpine Linux
│   └── LV : apps
│
├── LXC - Crafty
│   ├── Debian 13
│   └── LV : fast
│
├── VM - Windows
│   ├── GPU passthrough
│   └── LV : fast
│
├── LXC - Pi-hole
│   ├── Alpine Linux
│   └── LV : apps
│
└── LXC - Python Services
    ├── Alpine Linux
    └── LV : local-lvm
```

---

## Prévu

Services, changements ou améliorations dont le déploiement est prévu.

- [ ] Gitea
- [ ] VM Gitea Runner 
- [ ] BeamMP Server 
- [ ] Nextcloud
- [ ] Grafana
- [ ] Page web

--- 

## À réfléchir

Idées ou services potentiels qui nécessitent encore réflexion, tests ou validation.

- [ ] À définir

---

## Notes d'architecture

- Les services sont principalement isolés dans des **conteneurs LXC**.
- **Alpine Linux** est privilégié pour les services légers.
- **Nginx** et **Cloudflared** partagent actuellement le même conteneur LXC.
- Les workloads nécessitant une virtualisation complète sont exécutés dans des **VM**, notamment Windows avec **GPU passthrough**.
- Les informations relatives aux disques et au stockage sont documentées dans [`Disk.md`](./Disk.md).
