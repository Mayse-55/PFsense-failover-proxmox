# Configuration d'un cluster pfSense en haute disponibilité sous Proxmox

[![Proxmox](https://img.shields.io/badge/Proxmox-9.1.4-orange)](https://www.proxmox.com/)
[![pfSense](https://img.shields.io/badge/pfSense-2.8.1_Release-blue)](https://www.pfsense.org/)

**Guide technique pour la configuration d'un cluster pfSense en haute disponibilité avec synchronisation d'état sous Proxmox VE utilisant Open vSwitch.**

---

## Table des matières
1. [Prérequis](#prérequis)
2. [Architecture](#architecture)
3. [Installation](#installation)
4. [Configuration Open vSwitch](#configuration-open-vswitch)
5. [Configuration pfSense](#configuration-pfsense)
6. [Dépannage](#dépannage)
7. [Scripts utiles](#scripts-utiles)
8. [Ressources](#ressources)

---

## Prérequis

### Matériel et Logiciels
- **Proxmox VE** : Version 9.1.4 (testée et validée)
- **pfSense** : Version 2.8.1-RELEASE
- **Accès** : Droits administrateur (sudo/root)
- **Cluster** : Minimum 2 nœuds Proxmox

### Configuration Réseau
- 2 adresses IP publiques pour les interfaces WAN
- 1 sous-réseau dédié pour le LAN
- Connexion réseau stable et redondante entre les nœuds

> [!caution]
> Ce guide a été testé et validé sur des serveurs Proxmox **9.1.4** et pfSense **2.8.1-RELEASE**.
> La configuration utilise une carte réseau physique et une interface virtuelle OVS.
> En cas de dysfonctionnement, vérifiez la conformité de votre configuration système et réseau.

---

## Installation

### 1. Installation de pfSense

#### 1.1 Téléchargement de l'ISO
[Lien de téléchargement de l'ISO pfSense](https://github.com/ipsec-dev/pfsense-iso/releases)

#### 1.2 Création des machines virtuelles pfSense
Sur chaque serveur Proxmox, créer une VM avec les paramètres suivants :

| **Paramètre**       | **Valeur**                     |
|---------------------|--------------------------------|
| Type                | Linux 6.x - 2.6 Kernel         |
| Mémoire             | 4096 Mo                        |
| Processeurs         | 2 (Type: host)                 |
| Disque              | 32 Go (SCSI, cache: writeback) |
| Réseau              | virtio (bridge: vmbr0)         |

> Une VM pfSense doit être créée sur chaque nœud du cluster.

---

### 2. Installation des paquets nécessaires

#### 2.1 Installation d'Open vSwitch
```bash
# Mise à jour du système
apt update && apt dist-upgrade -y

# Installation d'Open vSwitch
apt install openvswitch-switch -y
```

#### 2.2 Configuration des bridges OVS
Sur chaque nœud Proxmox :
1. Accéder à l'interface web Proxmox
2. Naviguer vers **System** → **Network**
3. Créer un nouveau bridge OVS :
   - **Nom** : `vmbr1`

#### 2.3 Ajout des interfaces réseau aux VM
Pour chaque VM pfSense :
1. Éteindre la VM
2. Ajouter un périphérique réseau :
   - **Modèle** : VirtIO (paravirtualized)
   - **Bridge** : vmbr1
   - **VLAN Tag** : (selon votre configuration)
   - **Firewall** : Désactivé (selon votre configuration)
3. Démarrer la VM

#### Configuration des interfaces réseau :
```
pfSense1 - WAN : 192.168.1.101 
pfSense2 - WAN : 192.168.1.102 
IP virtuelle WAN : 192.168.1.110 (à configurer dans pfSense)

pfSense1 - LAN : 172.16.0.1 
pfSense2 - LAN : 172.16.0.2 
IP virtuelle LAN : 172.16.0.10 (à configurer dans pfSense)
```
> [!warning]
> Les IPs sont à adapter selon votre configuration

---

## Configuration Open vSwitch

### 3.1 Création du tunnel VXLAN

**Sur Proxmox 1 (192.168.1.101) :**
```bash
ovs-vsctl add-port vmbr1 vxlan-lan \
  -- set interface vxlan-lan type=vxlan \
     options:remote_ip=192.168.1.102 \
     options:key=2000
```

**Sur Proxmox 2 (192.168.1.102) :**
```bash
ovs-vsctl add-port vmbr1 vxlan-lan \
  -- set interface vxlan-lan type=vxlan \
     options:remote_ip=192.168.1.101 \
     options:key=2000
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

---

## Configuration pfSense

### 4.1 Configuration initiale

#### Installation via console :
1. Démarrer la VM avec l'ISO pfSense
2. Suivre l'assistant d'installation
3. Sélectionner **Auto (UFS)** pour le partitionnement
4. Configurer les interfaces réseaux

### 4.2 Configuration CARP

#### Sur pfSense 1 (Master) :

1. **Configuration des IP virtuelles**
   - Naviguer vers **Firewall** → **Virtual IPs** → **Add**
   - **Type** : CARP
   - **Interface** : LAN
   - **Adresse(s)** : 172.16.0.10 /24
   - **Mot de passe VIP** : [votre_mot_de_passe]
   - **VHID Group** : 1
   - **Advertising frequency** : 1 (base) & 0 (Skew)

   ![CARP LAN](https://github.com/user-attachments/assets/1660b640-fe6e-4742-9d03-43c09c30fa02)

2. **Répéter pour l'interface WAN** :
   - **Type** : CARP
   - **Interface** : WAN
   - **Adresse(s)** : 192.168.1.110 /24
   - **Mot de passe VIP** : [votre_mot_de_passe]
   - **VHID Group** : 1
   - **Advertising frequency** : 1 (base) & 0 (Skew)

   ![CARP WAN](https://github.com/user-attachments/assets/a9773d18-76c9-4526-9662-ed5099845a79)

#### Sur pfSense 2 (Backup) :
Répéter les étapes ci-dessus en modifiant **Skew** à **1**.

3. **Vérification du statut CARP**
   - Naviguer vers **Status** → **CARP (failover)**
   - Les deux adresses doivent apparaître avec le statut "Master" sur pfSense 1.

---

### 4.3 Configuration du NAT

1. Naviguer vers **Firewall** → **NAT**
2. Dans l'onglet **Outbound**, cocher **"Hybrid Outbound NAT rule generation"**
3. Indiquer le réseau source et l'adresse de translation.

![NAT Outbound](https://github.com/user-attachments/assets/f8d9fc5b-f0e5-4845-b257-857c1248a51c)

> **Note :**  
> Cette configuration n'est à effectuer que sur le pfSense primaire.  
> La réplication automatique se chargera de dupliquer les règles sur le nœud secondaire.

---

### 4.4 Configuration de la synchronisation haute disponibilité

1. Naviguer vers **System** → **High Avail. Sync**
2. Cocher **"Synchronize states"**
3. Indiquer :
   - **Interface de synchronisation** : LAN
   - **Adresse IP du pfSense 2** : [IP_LAN_pfSense2]
   - **Identifiants de connexion** : [vos_identifiants]
4. Cocher toutes les options de synchronisation disponibles.

![High Avail. Sync](https://github.com/user-attachments/assets/455ffb75-6078-4e61-b4c0-7dbea329be98)

---

### 4.5 Configuration des règles de pare-feu pour la réplication

1. Naviguer vers **Firewall** → **Rules**
2. Ajouter une règle pour autoriser la synchronisation :
   - **Source** : IP virtuelle LAN
   - **Destination** : This Firewall (self)
   - **Port** : 443 (HTTPS)
3. Ajouter une règle pour PFSYNC :
   - **Protocole** : PFSYNC
   - **Source** : IP virtuelle LAN
   - **Destination** : This Firewall (self)

![Règle HTTPS](https://github.com/user-attachments/assets/48fd975c-393c-4ff3-bc8e-998d3025a083)
![Règle PFSYNC](https://github.com/user-attachments/assets/1525e61e-816a-455a-a3c1-a3d6b788ed3c)

---

## 📋 Aide : Commandes de base Open vSwitch

### Voir la configuration complète OVS
```bash
ovs-vsctl show
```

### Lister tous les bridges
```bash
ovs-vsctl list-br
```

### Lister les ports d'un bridge
```bash
ovs-vsctl list-ports vmbr1
```

### Voir les détails d'une interface
```bash
ovs-vsctl list interface vxlan-lan
```

### Créer un tunnel VXLAN
#### Méthode 1 : Commande simple
```bash
# Créer un tunnel VXLAN sur vmbr1 vers un Proxmox distant
ovs-vsctl add-port vmbr1 vxlan-lan \
  -- set interface vxlan-lan type=vxlan \
     options:remote_ip=192.168.25.103 \
     options:key=2000
```
- **vmbr1** : Le bridge sur lequel attacher le tunnel
- **vxlan-lan** : Nom du tunnel (modifiable)
- **remote_ip** : IP du Proxmox distant
- **key** : Identifiant VXLAN (doit être identique des deux côtés)

#### Méthode 2 : Avec options avancées
```bash
# Tunnel VXLAN avec plus d'options
ovs-vsctl add-port vmbr1 vxlan-lan \
  -- set interface vxlan-lan type=vxlan \
     options:remote_ip=192.168.25.103 \
     options:key=2000 \
     options:dst_port=4789 \
     options:ttl=64
```
- **dst_port** : Port UDP VXLAN (défaut : 4789)
- **ttl** : Time To Live des paquets

### Exemples de tunnels multiples
```bash
# Tunnel LAN (vmbr1)
ovs-vsctl add-port vmbr1 vxlan-lan \
  -- set interface vxlan-lan type=vxlan \
     options:remote_ip=192.168.25.103 \
     options:key=2000

# Tunnel SYNC (vmbr2)
ovs-vsctl add-port vmbr2 vxlan-sync \
  -- set interface vxlan-sync type=vxlan \
     options:remote_ip=192.168.25.103 \
     options:key=3000

# Tunnel vers un 3ème Proxmox
ovs-vsctl add-port vmbr1 vxlan-to-pxm4 \
  -- set interface vxlan-to-pxm4 type=vxlan \
     options:remote_ip=192.168.25.104 \
     options:key=2000
```

### Supprimer un tunnel VXLAN
```bash
# Supprimer un tunnel spécifique
ovs-vsctl del-port vmbr1 vxlan-lan

# Supprimer TOUS les ports d'un bridge (ATTENTION)
# Liste d'abord les ports
ovs-vsctl list-ports vmbr1

# Supprime tous les ports un par un
ovs-vsctl del-port vmbr1 vxlan-lan
ovs-vsctl del-port vmbr1 vxlan-sync

# ⚠️ Supprimer un bridge complet (supprime le bridge ET tous ses tunnels)
ovs-vsctl del-br vmbr1
```

### Vérifier si un tunnel fonctionne
1. **Vérifier l'état de l'interface**
   ```bash
   ovs-vsctl list interface vxlan-lan | grep -E "link_state|error"
   ```
   - **Résultat attendu :**
     ```
     error               : []
     link_state          : up
     ```
     ✅ `error : []` = Pas d'erreur
     ✅ `link_state : up` = Interface active

2. **Voir les statistiques du tunnel**
   ```bash
   ovs-vsctl list interface vxlan-lan | grep statistics
   ```
   - **Résultat :**
     ```
     statistics          : {collisions=0, rx_bytes=125840, rx_crc_err=0, rx_dropped=0, rx_errors=0, rx_frame_err=0, rx_over_err=0, rx_packets=1520, tx_bytes=98560, tx_dropped=0, tx_errors=0, tx_packets=1240}
     ```
     ✅ `rx_packets` et `tx_packets` > 0 = Le trafic passe
     ❌ Si tous à 0 = Aucun trafic

3. **Voir toutes les infos du tunnel**
   ```bash
   ovs-vsctl list interface vxlan-lan
   ```
   - **Infos importantes à vérifier :**
     ```
     admin_state         : up
     error               : []
     ifindex             : 12
     link_state          : up
     options             : {key="2000", remote_ip="192.168.25.103"}
     status              : {tunnel_egress_iface="enp8s0", tunnel_egress_iface_carrier=up}
     type                : vxlan
     ```
     ✅ `tunnel_egress_iface_carrier=up` = Interface physique active
     ✅ `options` = Vérifiez que `remote_ip` et `key` sont corrects

4. **Tester avec tcpdump**
   ```bash
   # Voir le trafic VXLAN (port UDP 4789)
   tcpdump -i enp8s0 -n port 4789

   # Capturer le trafic sur le bridge virtuel
   tcpdump -i vmbr1 -n
   ```
   - **Test :** Effectuez un ping depuis un pfSense et observez le trafic dans tcpdump.

---

## ✅ Validation finale

À ce stade, votre cluster pfSense en haute disponibilité sous Proxmox devrait être pleinement opérationnel :
- Les deux nœuds pfSense sont synchronisés via CARP et PFSYNC.
- Les tunnels VXLAN sont actifs et permettent la communication entre les nœuds Proxmox.
- Les règles de pare-feu et de NAT sont répliquées automatiquement.
- Le basculement (failover) est testé et fonctionnel.

**Pour valider le bon fonctionnement :**
1. Vérifiez le statut CARP sur les deux nœuds pfSense.
2. Testez le basculement manuel en éteignant le nœud maître.
3. Vérifiez la connectivité réseau depuis le LAN et le WAN.
4. Consultez les logs système et les statistiques OVS pour détecter d’éventuelles anomalies.

En cas de problème, reportez-vous à la section [Dépannage](#dépannage) ou consultez les [Ressources](#ressources) pour obtenir de l’aide supplémentaire.
