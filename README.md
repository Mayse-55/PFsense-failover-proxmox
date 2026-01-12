# Configuration d'un cluster pfSense en haute disponibilité

[![Proxmox](https://img.shields.io/badge/Proxmox-9.1.4-orange)](https://www.proxmox.com/)
[![pfSense](https://img.shields.io/badge/pfSense-2.8.1_Release-blue)](https://www.pfsense.org/)

Guide technique pour la mise en place d'un cluster pfSense en haute disponibilité avec synchronisation d'état sous Proxmox VE utilisant Open vSwitch.

---

## Table des matières

1. [Prérequis](#prérequis)
2. [Architecture](#architecture)
3. [Installation](#installation)
4. [Configuration Open vSwitch](#configuration-open-vswitch)
5. [Configuration pfSense](#configuration-pfsense)
6. [Validation](#validation)
7. [Référence des commandes](#référence-des-commandes)

---

## Prérequis

### Environnement

- **Proxmox VE** : Version 9.1.4 ou supérieure
- **pfSense** : Version 2.8.1-RELEASE
- **Accès** : Droits administrateur root
- **Infrastructure** : Minimum 2 nœuds Proxmox

### Réseau

- Deux adresses IP pour les interfaces WAN
- Un sous-réseau dédié pour le LAN
- Connectivité réseau entre les nœuds Proxmox

> [!caution]  
> Ce guide a été testé avec **Proxmox VE 9.1.4** et **pfSense 2.8.1-RELEASE**.  
> Adaptez les adresses IP selon votre environnement.  

---

## Architecture

### Principe de fonctionnement

Le cluster repose sur plusieurs technologies :

- **CARP** : Protocole de redondance pour les adresses IP virtuelles
- **pfsync** : Synchronisation des tables d'états en temps réel
- **XMLRPC** : Réplication de la configuration
- **Open vSwitch** : Infrastructure de commutation virtuelle
- **VXLAN** : Tunnel réseau de niveau 2 entre les nœuds

### Schéma réseau

```
Internet
   │
   └─── WAN ───┬─── pfSense Master (192.168.1.101)
               ├─── pfSense Backup (192.168.1.102)
               └─── IP Virtuelle (192.168.1.110) ← CARP
   
   └─── LAN ───┬─── pfSense Master (172.16.0.1)
               ├─── pfSense Backup (172.16.0.2)
               └─── IP Virtuelle (172.16.0.10) ← CARP
```

---

## Installation

### 1. Création des machines virtuelles pfSense

### Téléchargement de l'ISO

[Télécharger pfSense](https://github.com/ipsec-dev/pfsense-iso/releases)

### Paramètres des VMs

Créer une VM sur chaque nœud Proxmox avec les caractéristiques suivantes :

| Paramètre | Valeur |
|-----------|--------|
| **OS Type** | Linux 6.x - 2.6 Kernel |
| **RAM** | 4096 MB |
| **CPU** | 2 cores (type: host) |
| **Disque** | 32 GB (SCSI, cache: writeback) |
| **Réseau** | VirtIO (bridge: vmbr0) |

> Une VM doit être créée sur chaque nœud du cluster.

#### Installation du système

1. Démarrer la VM avec l'ISO pfSense
2. Suivre l'assistant d'installation
3. Sélectionner le partitionnement **Auto (UFS)**
4. Configurer les interfaces réseau (WAN et LAN)
5. Redémarrer la VM

---

### 2. Installation d'Open vSwitch

Sur chaque nœud Proxmox, exécuter :

```bash
# Mise à jour du système
apt update && apt dist-upgrade -y

# Installation d'Open vSwitch
apt install openvswitch-switch -y

# Vérification
systemctl status openvswitch-switch
```

### 3. Création du bridge OVS

### Via l'interface web Proxmox

1. Accéder à **System** → **Network**
2. Cliquer sur **Create** → **OVS Bridge**
3. Configurer :
   - **Name** : `vmbr1`
   - **Autostart** : Coché
4. Appliquer la configuration

### 4. Ajout d'une interface réseau aux VMs

Pour chaque VM pfSense :

1. Éteindre la VM
2. Ajouter une interface réseau :
   - **Bridge** : vmbr1
   - **Model** : VirtIO
3. Démarrer la VM

---

## Configuration Open vSwitch

### Plan d'adressage

Définir un plan d'adressage cohérent :

**Interface WAN** :
```
pfSense Master  : 192.168.1.101
pfSense Backup  : 192.168.1.102
IP Virtuelle    : 192.168.1.110 (CARP)
```

**Interface LAN** :
```
pfSense Master  : 172.16.0.1
pfSense Backup  : 172.16.0.2
IP Virtuelle    : 172.16.0.10 (CARP)
```

> Les adresses IP sont à adapter selon votre configuration.

### Création du tunnel VXLAN

### Sur Proxmox 1 (192.168.1.101)

```bash
ovs-vsctl add-port vmbr1 vxlan-lan \
  -- set interface vxlan-lan type=vxlan \
     options:remote_ip=192.168.1.102 \
     options:key=2000
```

### Sur Proxmox 2 (192.168.1.102)

```bash
ovs-vsctl add-port vmbr1 vxlan-lan \
  -- set interface vxlan-lan type=vxlan \
     options:remote_ip=192.168.1.101 \
     options:key=2000
```

### Vérification du tunnel

```bash
# Voir la configuration
ovs-vsctl show

# Vérifier l'état
ovs-vsctl list interface vxlan-lan | grep -E "link_state|error"
```

**Résultat attendu** :
```
error               : []
link_state          : up
```

---

## Configuration pfSense

### 1.1 Configuration des interfaces

Se connecter à l'interface web de pfSense (identifiants par défaut : admin / pfsense).

### Interface WAN (Master)

- **Type** : Static IPv4
- **Adresse** : 192.168.1.101/24
- **Passerelle** : Votre gateway WAN

### Interface LAN (Master)

- **Type** : Static IPv4
- **Adresse** : 172.16.0.1/24

### 1.2 Répéter sur le PFsense (Backup) avec les adresses correspondantes.

Se connecter à l'interface web de pfSense (identifiants par défaut : admin / pfsense).

### Interface WAN (Backup)

- **Type** : Static IPv4
- **Adresse** : 192.168.1.102/24
- **Passerelle** : Votre gateway WAN

### Interface LAN (Backup)

- **Type** : Static IPv4
- **Adresse** : 172.16.0.2/24

---

### 2. Configuration CARP

### Sur pfSense Master

**IP Virtuelle LAN** :

1. Naviguer vers **Firewall** → **Virtual IPs** → **Add**
2. Configuration :
   - **Type** : CARP
   - **Interface** : LAN
   - **Address** : 172.16.0.10 / 24
   - **Virtual IP Password** : Mot de passe sécurisé
   - **VHID Group** : 1
   - **Advertising Frequency** : Base 1, Skew 0

![CARP LAN](https://github.com/user-attachments/assets/1660b640-fe6e-4742-9d03-43c09c30fa02)

**IP Virtuelle WAN** :

3. Cliquer sur **Add**
4. Configuration :
   - **Type** : CARP
   - **Interface** : WAN
   - **Address** : 192.168.1.110 / 24
   - **Virtual IP Password** : Même mot de passe
   - **VHID Group** : 2
   - **Advertising Frequency** : Base 1, Skew 0

![CARP WAN](https://github.com/user-attachments/assets/a9773d18-76c9-4526-9662-ed5099845a79)

### Sur pfSense Backup 
Répéter les étapes ci-dessus en modifiant **Skew** à **1**.

### Vérification du statut CARP
- Naviguer vers **Status** → **CARP (failover)**
- Les deux adresses doivent apparaître avec le statut "Master" sur pfSense 1.

---

### 3. Configuration NAT

Sur le pfSense Master uniquement :

1. Naviguer vers **Firewall** → **NAT** → **Outbound**
2. Sélectionner **Hybrid Outbound NAT rule generation**
3. Sauvegarder

![NAT Outbound](https://github.com/user-attachments/assets/f8d9fc5b-f0e5-4845-b257-857c1248a51c)

> La configuration sera automatiquement répliquée vers le Backup.

---

### 4. Synchronisation haute disponibilité

Sur le pfSense Master :

1. Naviguer vers **System** → **High Avail. Sync**
2. Configuration :
   - **Synchronize States** : Activé
   - **Synchronize Interface** : LAN
   - **pfsync Synchronize Peer IP** : 172.16.0.2
   - **Synchronize Config to IP** : 172.16.0.2
   - **Remote System Username** : admin
   - **Remote System Password** : Mot de passe du Backup
3. Cocher toutes les options de synchronisation
4. Sauvegarder

![High Avail. Sync](https://github.com/user-attachments/assets/455ffb75-6078-4e61-b4c0-7dbea329be98)

---

### 5. Règles de pare-feu

### Règle HTTPS (synchronisation XMLRPC)

1. Naviguer vers **Firewall** → **Rules** → **LAN** → **Add**
2. Configuration :
   - **Action** : Pass
   - **Protocol** : TCP
   - **Source** : 172.16.0.2
   - **Destination** : This firewall (self)
   - **Destination Port** : HTTPS (443)

![Règle HTTPS](https://github.com/user-attachments/assets/48fd975c-393c-4ff3-bc8e-998d3025a083)

### Règle pfsync

3. Cliquer sur **Add**
4. Configuration :
   - **Action** : Pass
   - **Protocol** : pfsync
   - **Source** : 172.16.0.2
   - **Destination** : This firewall (self)

![Règle PFSYNC](https://github.com/user-attachments/assets/1525e61e-816a-455a-a3c1-a3d6b788ed3c)

5. Sauvegarder et appliquer les changements

---

## Validation

### Test de synchronisation

1. Créer une règle de pare-feu sur le Master
2. Vérifier sa présence sur le Backup
3. Supprimer la règle de test

### Test de basculement

1. Depuis une machine du LAN, lancer un ping continu :
   ```bash
   ping -t 8.8.8.8
   ```

2. Arrêter la VM pfSense Master

3. Observer le basculement :
   - Le Backup passe en statut MASTER
   - Le ping continue sans interruption majeure

4. Redémarrer le Master :
   - Il reprend le statut MASTER
   - Le Backup redevient BACKUP

**Résultat attendu** : Basculement en moins de 5 secondes avec perte minimale de paquets.

---

## Référence des commandes

### Open vSwitch

#### Gestion des bridges

```bash
# Lister les bridges
ovs-vsctl list-br

# Créer un bridge
ovs-vsctl add-br vmbr1

# Supprimer un bridge
ovs-vsctl del-br vmbr1

# Lister les ports
ovs-vsctl list-ports vmbr1
```

#### Gestion des tunnels VXLAN

```bash
# Créer un tunnel
ovs-vsctl add-port vmbr1 vxlan-lan \
  -- set interface vxlan-lan type=vxlan \
     options:remote_ip=192.168.1.102 \
     options:key=2000

# Supprimer un tunnel
ovs-vsctl del-port vmbr1 vxlan-lan

# Modifier l'IP distante
ovs-vsctl set interface vxlan-lan options:remote_ip=192.168.1.103

# Voir les détails
ovs-vsctl list interface vxlan-lan
```

#### Diagnostic

```bash
# Configuration complète
ovs-vsctl show

# État de l'interface
ovs-vsctl list interface vxlan-lan | grep -E "link_state|error"

# Statistiques
ovs-vsctl list interface vxlan-lan | grep statistics

# Capture de trafic VXLAN
tcpdump -i enp8s0 -n port 4789
```

---

## Validation finale

Votre cluster pfSense en haute disponibilité est opérationnel lorsque :

- Les deux nœuds sont synchronisés via CARP et pfsync
- Les tunnels VXLAN sont actifs
- Les règles sont répliquées automatiquement
- Le basculement fonctionne sans interruption de service

**Tests de validation** :

1. Vérifier le statut CARP sur les deux nœuds
2. Tester le basculement en éteignant le Master
3. Vérifier la connectivité réseau depuis le LAN
4. Consulter les logs système

---

**Version** : 1.0  
**Dernière mise à jour** : Janvier 2026
