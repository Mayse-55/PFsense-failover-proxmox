# Configuration d'un cluster pfSense en HA sous Proxmox

![Proxmox](https://img.shields.io/badge/Proxmox-9.1.4-orange)
![pfSense](https://img.shields.io/badge/pfSense-2.8.1_Release-blue)
![Open vSwitch](https://img.shields.io/badge/Open_vSwitch-2.19-green)
![License](https://img.shields.io/badge/Documentation-CC--BY--4.0-lightgrey)

Guide complet pour configurer un cluster haute disponibilité pfSense avec synchronisation d'état sous Proxmox VE utilisant Open vSwitch.

## Table des matières
- [Prérequis](#prérequis)
- [Architecture](#architecture)
- [Installation](#installation)
- [Configuration Open vSwitch](#configuration-open-vswitch)
- [Configuration pfSense](#configuration-pfsense)
- [Dépannage](#dépannage)
- [Scripts utiles](#scripts-utiles)
- [Ressources](#ressources)

## Prérequis

### Matériel et Logiciels
- **Proxmox VE** : Version 9.1.4 (testé et validé)
- **pfSense** : Version 2.8.1-RELEASE
- **Accès** : Droits administrateur (sudo/root)
- **Cluster** : Minimum 2 nœuds Proxmox

### Configuration Réseau
- 2 adresses IP publiques pour les WAN
- 1 sous-réseau pour le LAN
- Connexion réseau stable entre les nœuds

> [!CAUTION]
> Cette documentation a été testée et validée sur des serveurs Proxmox **9.1.4** et deux pfSense **2.8.1-RELEASE**.
> Cette configuration utilise une carte réseau physique et une interface virtuelle OVS.
> En cas de problème, vérifiez votre configuration système et réseau.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Topologie Réseau                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌────────────┐                    ┌────────────┐         │
│  │  Proxmox 1 │◄──VXLAN Tunnel────►│  Proxmox 2 │         │
│  │ 192.168.1.10│      (vmbr1)      │192.168.1.20│         │
│  └─────┬──────┘                    └──────┬─────┘         │
│        │                                   │               │
│  ┌─────▼──────┐                    ┌──────▼─────┐         │
│  │ pfSense 1  │                    │ pfSense 2  │         │
│  │   MASTER   │                    │   BACKUP   │         │
│  └─────┬──────┘                    └──────┬─────┘         │
│        │                                   │               │
│    WAN │ LAN                          WAN │ LAN           │
└────────┼──────────────────────────────────┼───────────────┘
         ▼                                  ▼
    Internet                           Réseau Local
```

## Installation

### 1. Installation de pfSense

#### 1.1 Téléchargement de l'ISO
```bash
wget https://github.com/ipsec-dev/pfsense-iso/releases/download/2.8.1/pfSense-CE-memstick-2.8.1-RELEASE-amd64.iso
```

#### 1.2 Création des VM pfSense
Sur chaque serveur Proxmox :

| Paramètre | Valeur |
|-----------|---------|
| Type | Linux 6.x - 2.6 Kernel |
| Mémoire | 4096 MB |
| Processeurs | 2 (Type: host) |
| Disque | 32GB (SCSI, cache: writeback) |
| Réseau | virtio (bridge: vmbr0) |

> **Important** : Créez une VM pfSense sur chaque nœud du cluster.

### 2. Installation des paquets nécessaires

#### 2.1 Installation d'Open vSwitch
```bash
# Mise à jour du système
apt update && apt dist-upgrade -y

# Installation d'Open vSwitch
apt install openvswitch-switch openvswitch-common -y

# Vérification de l'installation
ovs-vsctl --version
```

#### 2.2 Configuration des bridges OVS
Sur Proxmox 1 et Proxmox 2 :

1. Accédez à l'interface web Proxmox
2. Naviguez vers **System** → **Network**
3. Cliquez sur **Create** → **OVS Bridge**
4. Configurez :
   - **Name** : `vmbr1`
   - **IPv4/CIDR** : `192.168.100.1/30` (Proxmox 1)
   - **IPv4/CIDR** : `192.168.100.2/30` (Proxmox 2)

#### 2.3 Ajout des interfaces réseau aux VM
Pour chaque VM pfSense :

1. Éteindre la VM
2. **Hardware** → **Add** → **Network Device**
3. Configuration :
   ```
   Model: VirtIO (paravirtualized)
   Bridge: vmbr1
   VLAN Tag: (selon votre configuration)
   Firewall: No
   ```
4. Démarrer la VM

## Configuration Open vSwitch

### 3.1 Création du tunnel VXLAN

**Sur Proxmox 1 (192.168.1.10) :**
```bash
ovs-vsctl add-port vmbr1 vxlan-lan \
  -- set interface vxlan-lan type=vxlan \
     options:remote_ip=192.168.1.20 \
     options:key=2000 \
     options:dst_port=4789
```

**Sur Proxmox 2 (192.168.1.20) :**
```bash
ovs-vsctl add-port vmbr1 vxlan-lan \
  -- set interface vxlan-lan type=vxlan \
     options:remote_ip=192.168.1.10 \
     options:key=2000 \
     options:dst_port=4789
```

### 3.2 Vérification de la configuration

#### Afficher la configuration OVS :
```bash
ovs-vsctl show
```

#### Vérifier l'état des interfaces :
```bash
ovs-vsctl list interface vxlan-lan | grep -E "link_state|error"
```

**Résultat attendu :**
```bash
error               : []
link_state          : up
```

#### Vérifier les statistiques VXLAN :
```bash
ovs-vsctl get interface vxlan-lan statistics
```

## Configuration pfSense

### 4.1 Configuration initiale

#### Installation via console :
1. Démarrer la VM avec l'ISO pfSense
2. Suivre l'assistant d'installation
3. Sélectionner **Auto (UFS)** pour le partitionnement
4. Redémarrer après installation

#### Configuration des interfaces :
```
WAN (wan)  -> vtnet0  -> Adresse publique
LAN (lan)  -> vtnet1  -> 192.168.1.1/24
SYNC (sync)-> vtnet2  -> 192.168.100.5/30
```

### 4.2 Configuration CARP (Haute Disponibilité)

#### Sur pfSense 1 (Master) :

1. **System** → **High Avail. Sync**
   - Synchronisation : Activée
   - Interface de synchronisation : SYNC
   - IP du partenaire : 192.168.100.6

2. **Services** → **CARP Settings**
   - Activer CARP
   - Virtual IPs :
     ```
     WAN VIP: 192.168.0.100/24
     LAN VIP: 192.168.1.254/24
     ```

3. **Firewall** → **Virtual IPs**
   - Ajouter les VIPs avec le même password CARP

#### Sur pfSense 2 (Backup) :

1. Répéter les étapes ci-dessus
2. **System** → **High Avail. Sync**
   - Mode : Synchroniser depuis le partenaire

### 4.3 Configuration de pfsync

```bash
# Éditer /etc/rc.conf.local sur les deux nœuds
cat >> /etc/rc.conf.local << EOF
pfsync_enable="YES"
pfsync_syncdev="sync0"
pfsync_syncpeer="192.168.100.6"  # Sur pfSense 1
# pfsync_syncpeer="192.168.100.5"  # Sur pfSense 2
EOF
```

## Dépannage

### Problèmes courants et solutions

#### 1. Tunnel VXLAN non fonctionnel
```bash
# Vérifier la connectivité
ping 192.168.1.20  # Depuis Proxmox 1 vers Proxmox 2

# Vérifier les logs
journalctl -u openvswitch-switch -f

# Redémarrer le service OVS
systemctl restart openvswitch-switch
```

#### 2. Synchronisation CARP échouée
```bash
# Vérifier l'état CARP
ifconfig | grep carp

# Vérifier pfsync
pfctl -s info | grep sync

# Tester la connectivité entre pfSense
ping 192.168.100.6  # Depuis pfSense 1
```

#### 3. Problèmes de performance réseau
```bash
# Optimiser les paramètres OVS
ovs-vsctl set Open_vSwitch . other_config:vlan-limit=0
ovs-vsctl set bridge vmbr1 datapath_type=netdev

# Vérifier la MTU
ip link show vxlan-lan
```

## Scripts utiles

### Vérification complète du cluster
```bash
#!/bin/bash
echo "=== Vérification du cluster pfSense HA ==="
echo ""

echo "1. État Open vSwitch :"
ovs-vsctl show | grep -A5 "Bridge vmbr1"
echo ""

echo "2. Statistiques VXLAN :"
ovs-vsctl get interface vxlan-lan statistics
echo ""

echo "3. Test de connectivité :"
ping -c 3 192.168.1.20
echo ""

echo "4. Services OVS :"
systemctl status openvswitch-switch --no-pager
```

### Monitoring des performances
```bash
#!/bin/bash
watch -n 5 '
echo "=== Monitoring OVS ==="
echo "Traffic VXLAN :"
ovs-vsctl get interface vxlan-lan statistics | \
  grep -E "rx_packets|tx_packets|rx_bytes|tx_bytes"
echo ""
echo "État des bridges :"
ovs-vsctl list-br
'
```

## Ressources supplémentaires

- **Documentation officielle pfSense** : https://docs.netgate.com/pfsense/
- **Guide Open vSwitch** : https://docs.openvswitch.org/
- **Proxmox VE Documentation** : https://pve.proxmox.com/wiki/Main_Page
- **Tutoriel GoOpenSource** : https://goopensource.fr/cluster-de-pfsense-en-failover/

## Notes importantes

### Bonnes pratiques
1. **Sauvegardes régulières** des configurations
2. **Test de basculement** planifié régulièrement
3. **Monitoring** des interfaces et du trafic
4. **Journalisation** détaillée pour le dépannage

### Sécurité
- Changer les mots de passe par défaut
- Configurer le firewall restrictivement
- Mettre à jour régulièrement les systèmes
- Utiliser des clés SSH pour l'authentification

### Maintenance
```bash
# Mise à jour régulière
apt update && apt upgrade -y

# Nettoyage des logs
logrotate -f /etc/logrotate.conf

# Vérification de l'espace disque
df -h
```

---

## License

Cette documentation est mise à disposition sous licence Creative Commons Attribution 4.0 International.

**Dernière mise à jour** : Décembre 2024  
**Version du document** : 1.0  

---

> **Remarque** : Cette configuration a été testée en environnement de production. Toujours tester dans un environnement de développement avant le déploiement en production.

**Étapes suivantes** :
- [ ] Configurer le monitoring (Zabbix/PRTG)
- [ ] Mettre en place les sauvegardes automatiques
- [ ] Documenter la procédure de restauration
- [ ] Planifier les tests de basculement
