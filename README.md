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

> **Remarque** :  
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

> **Note**
> Cette configuration n'est à effectuer que sur le pfSense primaire. La réplication automatique se chargera de dupliquer les règles sur le nœud secondaire.

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
