# 🔐 Self-Hosted VPN Infrastructure with Centralized Auth & DNS Filtering (ISP Maroc Telecom)

> Infrastructure réseau auto-hébergée complète — hyperviseur bare-metal, VPN multi-protocoles, authentification centralisée RADIUS et filtrage DNS défensif.

![Status](https://img.shields.io/badge/Status-Production-brightgreen)
![Uptime](https://img.shields.io/badge/Uptime-99.9%25-blue)
![Users](https://img.shields.io/badge/Users-5-orange)
![DNS](https://img.shields.io/badge/DNS-40k%20req%2Fjour-purple)

---

## 🎯 Contexte et objectifs

Ce projet est né d'une contrainte réelle : déployer une infrastructure réseau sécurisée depuis un réseau domestique avec une **IP dynamique**, sans accès au panneau opérateur, pour des utilisateurs non-techniques.

**Contraintes imposées :**
- IP publique dynamique changeant toutes les 24h (Maroc Telecom FTTH)
- Matériel limité : mini PC fanless
- Utilisateurs finaux non-techniques → plug-and-play obligatoire
- Budget nul → 100% open source

**Objectifs atteints :**
- Hyperviseur Type-1 stable, administrable à distance
- Accès VPN sécurisé depuis n'importe où dans le monde
- Authentification centralisée RADIUS — scalable pour de futurs services
- Filtrage DNS (publicités, malwares, gambling)
- Isolation complète des clients VPN du réseau local

---

## 🏗️ Architecture globale

```
Internet
    │
    ▼
┌─────────────────────────────────────────┐
│           Maroc Telecom FTTH            │
│           ZTE F6600 (OLT)               │
│           IP Publique Dynamique         │
└──────────────────┬──────────────────────┘
                   │ Port Forwarding
                   ▼
┌─────────────────────────────────────────┐
│           Proxmox VE (Type-1)           │
│           10.0.1.180/21                 │
│                                         │
│  ┌─────────┐ ┌──────────┐ ┌──────────┐ │
│  │LXC      │ │VM        │ │LXC       │ │
│  │DDNS     │ │SoftEther │ │FreeRADIUS│ │
│  │ddclient │ │VPN Server│ │Auth      │ │
│  │         │ │10.0.1.181│ │10.0.1.182│ │
│  └─────────┘ └──────────┘ └──────────┘ │
│                                         │
│  Réseau segmenté /21 :                  │
│  10.0.0.0/24  →  Équipements réseau    │
│  10.0.1.0/24  →  Infra & serveurs      │
│  10.0.2.0/24  →  WiFi & accès         │
│  10.0.3.0/24  →  Domotique & caméras  │
└─────────────────────────────────────────┘
                   │ DNS queries
                   ▼
┌─────────────────────────────────────────┐
│         Oracle Cloud (Always Free)      │
│         Pi-hole + dnsmasq               │
│         dns.example.com — X.X.X.X   │
└─────────────────────────────────────────┘
```

---

## 🛠️ Stack technique

| Composant | Technologie | Rôle |
|-----------|-------------|------|
| Hyperviseur | Proxmox VE 8.x (Type-1) | Virtualisation bare-metal |
| Dynamic DNS | LXC + ddclient + API Namecheap | Résolution IP dynamique |
| Serveur VPN | VM SoftEther VPN | Multi-protocoles |
| Auth centralisée | LXC FreeRADIUS | Gestion utilisateurs unifiée |
| DNS Filtrant | Pi-hole + dnsmasq (Oracle Cloud) | Blocage pub/malware |
| Firewall | iptables | Filtrage par blocs IP régionaux |

---

## 📦 Étape 1 — Proxmox : la fondation

### Pourquoi Proxmox
Proxmox VE est un hyperviseur Type-1 open source. Il tourne directement sur le hardware sans OS hôte, maximisant performances et stabilité. Il supporte les VMs (isolation complète) et les LXC (conteneurs légers partageant le kernel — idéal pour des services légers comme DDNS ou RADIUS).

### Conception du réseau
Le réseau domestique a été redesigné en **/21** — assez large pour segmenter proprement les usages, sans le gaspillage d'un /16 :

```
10.0.0.0/21  →  Plage globale (2046 hôtes)
10.0.0.0/24  →  Équipements réseau (box, switches)
10.0.1.0/24  →  Infra & serveurs (Proxmox, VMs, LXCs)
10.0.2.0/24  →  WiFi & points d'accès
10.0.3.0/24  →  Domotique (télés, caméras IP)
```

> ⚠️ La box opérateur ne supportant pas les VLANs, la segmentation est logique — gérée au niveau Proxmox via les bridges réseau.

### Déploiement plug-and-play
Le mini PC est préconfiguré avec l'IP statique `10.0.1.180` avant envoi. Les utilisateurs branchent le câble Ethernet — zéro intervention de leur côté.

---

## 🌐 Étape 2 — Dynamic DNS : résoudre l'IP dynamique

### Problème
Maroc Telecom attribue une IP publique dynamique changeant toutes les 24h. Sans mécanisme de résolution, impossible de se connecter au VPN de façon fiable.

### Solution
Un LXC Proxmox léger fait tourner **ddclient** en permanence, interrogeant l'API Namecheap toutes les 5 minutes pour mettre à jour l'enregistrement DNS.

```bash
# ddclient.conf (simplifié)
protocol=namecheap
server=dynamicdns.park-your-domain.com
login=example.com
password=<API_KEY>
akram
```

```
vpn.example.com  →  pointe toujours vers l'IP publique courante
```

**Résultat :** un hostname stable comme unique point d'entrée, quelle que soit l'IP du moment.

---

## 🔑 Étape 3 — SoftEther VPN : le serveur

### Pourquoi SoftEther
SoftEther supporte nativement plusieurs protocoles VPN depuis un seul déploiement — tous les appareils clients sont couverts sans serveur supplémentaire :

| Protocole | Client cible |
|-----------|-------------|
| OpenVPN | Desktop (Windows, Linux, macOS) |
| L2TP/IPsec | Natif iOS & Android |
| MS-SSTP | Windows natif sans client tiers |
| SSL-VPN | Contournement firewalls restrictifs |

### Configuration
- Hub VPN Standalone
- SecureNAT activé pour le routage interne clients
- Port principal : 8080/UDP
- Chiffrement : AES-256

### Authentification déléguée
Les utilisateurs ne sont **pas** gérés localement dans SoftEther. Toute authentification est déléguée à FreeRADIUS — ce qui permet de centraliser la gestion des accès pour l'ensemble de l'infrastructure, pas seulement le VPN.

```
Client VPN  →  SoftEther  →  RADIUS Request  →  FreeRADIUS  →  Accept/Reject
```

---

## 👥 Étape 4 — FreeRADIUS : authentification centralisée

### Pourquoi RADIUS plutôt que la gestion locale
Gérer les utilisateurs directement dans SoftEther crée un silo : si demain un nouveau service (Wi-Fi WPA2-Enterprise, portail captif, autre VPN) nécessite une auth, il faut tout reconfigurer.

FreeRADIUS centralise **tous les accès en un point unique** — un utilisateur créé une fois, disponible pour tous les services compatibles RADIUS.

### Architecture

```
┌─────────────┐    Access-Request (port 1812)   ┌──────────────────┐
│  SoftEther  │ ──────────────────────────────► │   FreeRADIUS     │
│  VPN Server │ ◄────────────────────────────── │   LXC Proxmox    │
└─────────────┘    Access-Accept / Access-Reject │   10.0.1.182     │
                                                  └──────────────────┘
```

### En production
- 5 utilisateurs actifs avec credentials individuels
- Révocation instantanée sans toucher à la config VPN
- Logs d'authentification centralisés

### Vision long terme
FreeRADIUS est pensé comme la **colonne vertébrale d'authentification** de toute l'infrastructure — réutilisable pour tout futur service nécessitant un contrôle d'accès centralisé.

---

## 🛡️ Étape 5 — Pi-hole : filtrage DNS défensif

### Pourquoi Pi-hole
Ouvrir un accès VPN à plusieurs utilisateurs crée une responsabilité légale : **toutes leurs requêtes sortent sous mon IP publique.** Si un client visite un site illégal, télécharge du contenu protégé, ou accède à des domaines malveillants — c'est mon adresse IP qui apparaît dans les logs.

Pi-hole agit comme un **filtre DNS centralisé** : il bloque en amont les domaines dangereux, illégaux ou indésirables avant même que la connexion soit établie. Tous les clients VPN passent obligatoirement par ce filtre — impossible à contourner (voir étape 6).

### Pourquoi sur Oracle Cloud et pas sur Proxmox
SoftEther utilise le port 53 pour son mode VPN-over-DNS (tunneling dans les requêtes DNS pour contourner les firewalls restrictifs). Impossible d'ouvrir ce même port pour Pi-hole depuis la box — conflit direct.

**Solution :** Pi-hole déployé sur une VM Oracle Cloud Always Free — gratuit, IP fixe, disponible indépendamment de l'état du serveur Proxmox.

```
VM Oracle Cloud Ubuntu 20.04
IP publique : X.X.X.X
Accessible via : dns.example.com
```

### Listes de blocage

| Catégorie | Domaines bloqués |
|-----------|-----------------|
| Publicités | ~1 000 000 |
| Malwares & phishing | ~500 000 |
| Gambling & paris | ~50 000 |
| Streaming illégal | ~20 000 |

**Taux de blocage en production : ~23% des requêtes**

**Whitelist regex** — seuls les TLDs courants sont autorisés :
```regex
^([a-z0-9-]+\.)+\.(com|net|org|fr|de|ma|uk)$
```

---

## 🔒 Étape 6 — Sécurité : défense en profondeur

### Filtrage iptables par blocs IP AFRINIC
Pi-hole exposé publiquement attire immédiatement des bots et scanners. Plutôt que de tout bloquer (et casser le service), seuls les blocs IP attribués par AFRINIC à Maroc Telecom sont autorisés sur le port 53 — toutes les IPs dynamiques de Maroc Telecom appartiennent à ces blocs.

```bash
# Blocs IP Maroc Telecom (registre AFRINIC)
iptables -A INPUT -s 105.128.0.0/11 -p udp --dport 53 -j ACCEPT
iptables -A INPUT -s 196.217.0.0/16 -p udp --dport 53 -j ACCEPT
iptables -A INPUT -s 41.248.0.0/14  -p udp --dport 53 -j ACCEPT
iptables -A INPUT -p udp --dport 53  -j REJECT  # Drop tout le reste
```

### Redirection DNS forcée
Empêche les clients VPN de contourner le filtrage en changeant leur DNS manuellement ou via DoH natif des navigateurs :

```bash
# Redirige tout trafic port 53 vers Pi-hole — incontournable
iptables -t nat -A PREROUTING -p udp --dport 53 -j DNAT --to-destination 127.0.0.1

# Bloque DNS over TLS (port 853) — bypasse Pi-hole
iptables -A FORWARD -p tcp --dport 853 -j REJECT

# Bloque les DNS publics connus directement
iptables -A OUTPUT -d 8.8.8.8 -j REJECT
iptables -A OUTPUT -d 1.1.1.1 -j REJECT

# IPv6 désactivé — non supporté par l'infrastructure actuelle
ip6tables -P INPUT DROP
ip6tables -P FORWARD DROP
```

### Isolation des clients VPN
Les clients VPN sont cloisonnés — impossible d'accéder aux équipements du réseau local (caméras, télés, NAS) :

```
ACL SoftEther : DISCARD src=192.168.30.0/24 dst=10.0.0.0/21
```

### QoS — Rate limiting par client
```
Download max par client : 50 Mb/s
Upload max par client   : 20 Mb/s
Connexions TCP max      : 32 simultanées
```

---

## 📊 Performances en production

| Métrique | Valeur |
|----------|--------|
| Uptime | 99.9% — aucun incident depuis déploiement |
| Requêtes DNS/jour | ~40 000 |
| Taux de blocage DNS | ~23% |
| Utilisateurs actifs | 5 |
| Appareils simultanés max testés | 15 |
| Latence VPN moyenne | < 30ms |

---

## 🚧 Problèmes rencontrés et solutions

| Problème | Cause | Solution appliquée |
|----------|-------|--------------------|
| IP dynamique imprévisible | FAI change l'IP toutes les 24h | DDNS via ddclient + API Namecheap |
| Conflit port 53 | SoftEther utilise port 53 pour VPN-over-DNS | Migration Pi-hole sur Oracle Cloud |
| Bots scannant le DNS | Pi-hole exposé publiquement | Filtrage iptables par blocs AFRINIC |
| Contournement DNS | Navigateurs activent DoH nativement | Redirection port 53 + blocage port 853 |
| Accès réseau local depuis VPN | Pas d'isolation par défaut | ACL SoftEther — isolation complète |

---

## 🔮 Améliorations prévues

- [ ] Migration OpenVPN → WireGuard (latence et performances améliorées)
- [ ] Dashboard Grafana — monitoring temps réel (bande passante, connexions actives)
- [ ] Alertes Telegram sur tentatives d'intrusion ou déconnexions anormales
- [ ] Support IPv6 (en attente déploiement Maroc Telecom)
- [ ] Backup automatique configuration Proxmox vers stockage distant

---

*Infrastructure déployée et maintenue depuis 2025. Aucun incident de sécurité à ce jour.*
