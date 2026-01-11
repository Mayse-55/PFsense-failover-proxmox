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
6. [Validation et tests](#validation-et-tests)
7. [Dépannage](#dépannage)
8. [Référence des commandes](#référence-des-commandes)
9. [Ressources](#ressources)

---

## Prérequis

### Environnement matériel et logiciel
- **Proxmox VE** : Version 9.1.4 (testée et validée)
- **pfSense** : Version 2.8.1-RELEASE
- **Privilèges** : Accès administrateur (root/sudo)
- **Infrastructure** : Minimum 2 nœuds Proxmox en réseau

### Configuration réseau requise
- Deux adresses IP publiques pour les interfaces WAN
- Un sous-réseau dédié pour le réseau LAN
- Connectivité réseau stable et redondante entre les nœuds Proxmox
- Switch managé (optionnel mais recommandé pour VLAN)

> **Important** : Ce guide a été testé et validé sur Proxmox VE 9.1.4 et pfSense 2.8.1-RELEASE. L'architecture utilise Open vSwitch avec des tunnels VXLAN pour la communication inter-nœuds. Adaptez les adresses IP et configurations réseau selon votre environnement.

---

## Architecture

### Vue d'ensemble

Le cluster pfSense en haute disponibilité repose sur les composants suivants :

- **CARP (Common Address Redundancy Protocol)** : Protocole de redondance permettant le partage d'adresses IP virtuelles
- **pfsync** : Synchronisation en temps réel des tables d'états entre les nœuds
- **XMLRPC** : Synchronisation de la configuration entre le maître et le secondaire
- **Open vSwitch** : Solution de commutation virtuelle permettant les tunnels VXLAN
- **VXLAN** : Tunnel de couche 2 pour interconnecter les bridges réseau entre nœuds

### Schéma réseau

```
Internet
   │
   ├──────────────┬──────────────┐
   │              │              │
┌──▼───────┐  ┌──▼───────┐  ┌──▼───────┐
│  WAN 1   │  │  WAN 2   │  │ WAN VIP  │ (CARP)
│ Master   │  │ Backup   │  │ Failover │
└──┬───────┘  └──┬───────┘  └──┬───────┘
   │              │              │
   └──────────────┴──────────────┘
         Tunnel VXLAN
   ┌──────────────┬──────────────┐
   │              │              │
┌──▼───────┐  ┌──▼───────┐  ┌──▼───────┐
│  LAN 1   │  │  LAN 2   │  │  LAN VIP │ (CARP)
│ Master   │  │ Backup   │  │ Failover │
└──────────┘  └──────────┘  └──────────┘
```

---

## Installation

### 1. Préparation des machines virtuelles pfSense

#### 1.1 Téléchargement de l'image ISO

Télécharger l'image ISO officielle de pfSense depuis le dépôt GitHub :
[Télécharger pfSense ISO](https://github.com/pfsense/pfsense/releases)

#### 1.2 Création des machines virtuelles

Sur chaque nœud Proxmox, créer une machine virtuelle avec les spécifications suivantes :

| Paramètre              | Valeur recommandée              |
|------------------------|---------------------------------|
| **Système d'exploitation** | Linux 6.x - 2.6 Kernel      |
| **Mémoire RAM**        | 4096 MB (minimum 2048 MB)       |
| **Processeurs**        | 2 vCPU (Type: host)             |
| **Disque dur**         | 32 GB (SCSI, Cache: writeback)  |
| **Première interface** | VirtIO (Bridge: vmbr0)          |
| **Options**            | Start at boot: enabled          |

> **Note** : Une machine virtuelle pfSense doit être créée sur chaque nœud du cluster Proxmox.

#### 1.3 Installation du système pfSense

1. Monter l'image ISO sur la machine virtuelle
2. Démarrer la VM et suivre l'assistant d'installation
3. Sélectionner le type de partitionnement : **Auto (UFS)**
4. Configurer les interfaces réseau initiales (WAN et LAN)
5. Redémarrer la VM après l'installation

---

### 2. Configuration de l'infrastructure Open vSwitch

#### 2.1 Installation d'Open vSwitch

Sur chaque nœud Proxmox, exécuter les commandes suivantes :

```bash
# Mise à jour du système
apt update && apt dist-upgrade -y

# Installation d'Open vSwitch
apt install openvswitch-switch -y

# Vérification de l'installation
systemctl status openvswitch-switch
```

#### 2.2 Création des bridges OVS

**Via l'interface web Proxmox** :

1. Se connecter à l'interface web Proxmox
2. Naviguer vers le nœud concerné
3. Accéder à **System** → **Network**
4. Cliquer sur **Create** → **OVS Bridge**
5. Configurer les paramètres :
   - **Name** : `vmbr1`
   - **Autostart** : Activé
6. Appliquer la configuration

**Via la ligne de commande** :

```bash
# Créer le bridge OVS
ovs-vsctl add-br vmbr1

# Vérifier la création
ovs-vsctl list-br
```

#### 2.3 Ajout des interfaces réseau aux machines virtuelles

Pour chaque machine virtuelle pfSense :

1. Arrêter la VM
2. Accéder à **Hardware** → **Add** → **Network Device**
3. Configurer les paramètres :
   - **Bridge** : vmbr1
   - **Model** : VirtIO (paravirtualized)
   - **Firewall** : Désactivé
   - **VLAN Tag** : (optionnel, selon configuration)
4. Démarrer la VM

---

## Configuration Open vSwitch

### 3.1 Plan d'adressage IP

Définir un plan d'adressage cohérent pour l'infrastructure :

**Exemple de configuration WAN** :
```
pfSense Master  : 192.168.1.101
pfSense Backup  : 192.168.1.102
IP Virtuelle WAN: 192.168.1.110 (CARP)
```

**Exemple de configuration LAN** :
```
pfSense Master  : 172.16.0.1
pfSense Backup  : 172.16.0.2
IP Virtuelle LAN: 172.16.0.10 (CARP)
```

> **Important** : Adapter ces adresses IP selon votre environnement réseau.

### 3.2 Création des tunnels VXLAN

Les tunnels VXLAN permettent d'interconnecter les bridges OVS entre les nœuds Proxmox.

**Sur le nœud Proxmox 1 (par exemple 192.168.1.101)** :

```bash
ovs-vsctl add-port vmbr1 vxlan-lan \
  -- set interface vxlan-lan type=vxlan \
     options:remote_ip=192.168.1.102 \
     options:key=2000
```

**Sur le nœud Proxmox 2 (par exemple 192.168.1.102)** :

```bash
ovs-vsctl add-port vmbr1 vxlan-lan \
  -- set interface vxlan-lan type=vxlan \
     options:remote_ip=192.168.1.101 \
     options:key=2000
```

**Paramètres** :
- `vmbr1` : Bridge OVS sur lequel créer le tunnel
- `vxlan-lan` : Nom de l'interface tunnel (personnalisable)
- `remote_ip` : Adresse IP du nœud Proxmox distant
- `key` : Identifiant VXLAN (doit être identique sur les deux nœuds)

### 3.3 Vérification de la configuration

#### Afficher la configuration complète OVS :

```bash
ovs-vsctl show
```

**Résultat attendu** :
```
Bridge vmbr1
    Port vmbr1
        Interface vmbr1
            type: internal
    Port vxlan-lan
        Interface vxlan-lan
            type: vxlan
            options: {key="2000", remote_ip="192.168.1.102"}
```

#### Vérifier l'état du tunnel :

```bash
ovs-vsctl list interface vxlan-lan | grep -E "link_state|error"
```

**Résultat attendu** :
```
error               : []
link_state          : up
```

#### Vérifier les statistiques du tunnel :

```bash
ovs-vsctl list interface vxlan-lan | grep statistics
```

Le trafic (`rx_packets` et `tx_packets`) doit être supérieur à 0 si le tunnel est actif.

### 3.4 Persistance de la configuration

Pour rendre la configuration VXLAN persistante après redémarrage, éditer le fichier `/etc/network/interfaces` :

```bash
nano /etc/network/interfaces
```

Ajouter la configuration suivante dans la section du bridge vmbr1 :

```bash
auto vmbr1
iface vmbr1 inet manual
    ovs_type OVSBridge
    ovs_ports vxlan-lan
    post-up ovs-vsctl --may-exist add-port vmbr1 vxlan-lan -- set interface vxlan-lan type=vxlan options:remote_ip=192.168.1.102 options:key=2000
```

Adapter l'adresse `remote_ip` selon le nœud.

---

## Configuration pfSense

### 4.1 Configuration des interfaces réseau

#### Accès à l'interface web

1. Se connecter à l'interface web de pfSense via l'adresse IP LAN
2. Identifiants par défaut :
   - **Utilisateur** : admin
   - **Mot de passe** : pfsense

#### Configuration des interfaces

Naviguer vers **Interfaces** et configurer les interfaces WAN et LAN selon le plan d'adressage défini.

**Interface WAN (pfSense Master)** :
- **IPv4 Configuration Type** : Static IPv4
- **IPv4 Address** : 192.168.1.101/24
- **Gateway** : Adresse de la passerelle WAN

**Interface LAN (pfSense Master)** :
- **IPv4 Configuration Type** : Static IPv4
- **IPv4 Address** : 172.16.0.1/24

Répéter la configuration sur le pfSense Backup avec les adresses correspondantes.

### 4.2 Configuration des adresses IP virtuelles (CARP)

CARP permet de créer des adresses IP virtuelles partagées entre les deux nœuds pfSense.

#### Sur le pfSense Master :

**Configuration IP virtuelle LAN** :

1. Naviguer vers **Firewall** → **Virtual IPs** → **Add**
2. Configurer les paramètres :
   - **Type** : CARP
   - **Interface** : LAN
   - **Address(es)** : 172.16.0.10 / 24
   - **Virtual IP Password** : Définir un mot de passe fort
   - **VHID Group** : 1
   - **Advertising Frequency** :
     - **Base** : 1
     - **Skew** : 0 (priorité maître)
   - **Description** : CARP VIP LAN

**Configuration IP virtuelle WAN** :

1. Cliquer sur **Add** pour créer une nouvelle VIP
2. Configurer les paramètres :
   - **Type** : CARP
   - **Interface** : WAN
   - **Address(es)** : 192.168.1.110 / 24
   - **Virtual IP Password** : Même mot de passe que pour LAN
   - **VHID Group** : 2
   - **Advertising Frequency** :
     - **Base** : 1
     - **Skew** : 0 (priorité maître)
   - **Description** : CARP VIP WAN

#### Sur le pfSense Backup :

Répéter les mêmes étapes de configuration en modifiant uniquement le paramètre **Skew** :
- **Skew** : 100 (priorité secondaire)

> **Note** : Le mot de passe CARP et le VHID Group doivent être identiques sur les deux nœuds.

### 4.3 Vérification du statut CARP

1. Naviguer vers **Status** → **CARP (failover)**
2. Vérifier l'état des adresses virtuelles :
   - Sur le Master : Statut **MASTER**
   - Sur le Backup : Statut **BACKUP**

### 4.4 Configuration de la synchronisation haute disponibilité

La synchronisation permet de répliquer automatiquement la configuration du Master vers le Backup.

#### Sur le pfSense Master uniquement :

1. Naviguer vers **System** → **High Avail. Sync**

**Configuration de la synchronisation d'état (pfsync)** :

- **Synchronize States** : Activé
- **Synchronize Interface** : LAN
- **pfsync Synchronize Peer IP** : 172.16.0.2 (IP LAN du Backup)

**Configuration de la synchronisation de configuration (XMLRPC)** :

- **Synchronize Config to IP** : 172.16.0.2
- **Remote System Username** : admin
- **Remote System Password** : Mot de passe du compte admin du Backup

**Options de synchronisation** :

Activer tous les éléments de configuration à synchroniser :
- Firewall Rules
- Firewall Schedules
- Firewall Aliases
- NAT
- IPsec
- OpenVPN
- Certificate Authorities
- Certificates
- DHCP Server
- Wake on LAN
- Static Routes
- Virtual IPs
- Traffic Shaper (Limiter)
- Traffic Shaper (Queues)

2. Cliquer sur **Save**

### 4.5 Configuration des règles de pare-feu

Pour permettre la synchronisation et la communication CARP, configurer les règles de pare-feu suivantes.

#### Règles sur l'interface LAN :

**Règle 1 : Autoriser XMLRPC (HTTPS)** :

1. Naviguer vers **Firewall** → **Rules** → **LAN** → **Add**
2. Configurer :
   - **Action** : Pass
   - **Protocol** : TCP
   - **Source** : Single host or alias → 172.16.0.2
   - **Destination** : This firewall (self)
   - **Destination Port Range** : HTTPS (443)
   - **Description** : Allow XMLRPC sync from Backup

**Règle 2 : Autoriser pfsync** :

1. Cliquer sur **Add**
2. Configurer :
   - **Action** : Pass
   - **Protocol** : pfsync
   - **Source** : Single host or alias → 172.16.0.2
   - **Destination** : This firewall (self)
   - **Description** : Allow pfsync from Backup

**Règle 3 : Autoriser CARP** :

1. Cliquer sur **Add**
2. Configurer :
   - **Action** : Pass
   - **Protocol** : CARP
   - **Source** : LAN net
   - **Destination** : LAN net
   - **Description** : Allow CARP advertisements

3. Cliquer sur **Save** et **Apply Changes**

> **Note** : Ces règles seront automatiquement synchronisées vers le Backup.

### 4.6 Configuration NAT Outbound

Pour permettre au réseau LAN d'accéder à Internet via l'adresse IP virtuelle WAN :

1. Naviguer vers **Firewall** → **NAT** → **Outbound**
2. Sélectionner **Hybrid Outbound NAT rule generation**
3. Cliquer sur **Save**
4. Ajouter une règle manuelle :
   - **Interface** : WAN
   - **Source** : 172.16.0.0/24 (réseau LAN)
   - **Translation** → **Address** : 192.168.1.110 (IP virtuelle WAN)
   - **Description** : NAT outbound via CARP VIP
5. Cliquer sur **Save** et **Apply Changes**

---

## Validation et tests

### 5.1 Vérification de la synchronisation

#### Test de synchronisation de configuration :

1. Sur le pfSense Master, créer une règle de pare-feu de test
2. Vérifier que la règle apparaît automatiquement sur le Backup
3. Supprimer la règle de test

#### Test de synchronisation d'état :

1. Établir une connexion depuis le LAN vers Internet (par exemple, un ping continu)
2. Sur le Master, naviguer vers **Diagnostics** → **States** → **States**
3. Vérifier la présence de la connexion active
4. Sur le Backup, vérifier que les mêmes états sont présents

### 5.2 Test de basculement (failover)

#### Procédure de test :

1. Depuis une machine du réseau LAN, lancer un ping continu vers une adresse Internet :
   ```bash
   ping -t 8.8.8.8
   ```

2. Sur le Master, vérifier le statut CARP : **Status** → **CARP (failover)**

3. Simuler une panne du Master :
   - Option 1 : Arrêter la VM pfSense Master
   - Option 2 : Désactiver l'interface WAN sur le Master
   - Option 3 : Utiliser la fonction **Enter Persistent CARP Maintenance Mode** dans **Status** → **CARP (failover)**

4. Observer le basculement :
   - Le Backup passe en statut **MASTER**
   - Le ping doit continuer sans interruption (ou perte minimale de 1-2 paquets)

5. Restaurer le Master :
   - Redémarrer la VM ou réactiver l'interface
   - Le Master reprend le statut **MASTER**
   - Le Backup repasse en statut **BACKUP**

#### Résultat attendu :

- Basculement automatique en moins de 5 secondes
- Perte de paquets minimale (< 3 paquets)
- Reprise automatique lorsque le Master revient en ligne

### 5.3 Vérification des logs

#### Sur les deux nœuds pfSense :

1. Naviguer vers **Status** → **System Logs** → **System**
2. Rechercher les entrées relatives à CARP et pfsync
3. Vérifier l'absence d'erreurs critiques

**Entrées de log normales lors du basculement** :
```
CARP: BACKUP -> MASTER on vlan100
pfsync: state synchronization successful
```

---

## Dépannage

### 6.1 Problèmes courants et solutions

#### Le tunnel VXLAN ne fonctionne pas

**Symptômes** :
- `link_state: down` dans la configuration OVS
- Pas de trafic entre les nœuds

**Vérifications** :

1. Connectivité réseau entre les nœuds Proxmox :
   ```bash
   ping [IP_nœud_distant]
   ```

2. Port UDP 4789 ouvert (port VXLAN par défaut) :
   ```bash
   nc -u -zv [IP_nœud_distant] 4789
   ```

3. Pare-feu Proxmox :
   ```bash
   iptables -I INPUT -p udp --dport 4789 -j ACCEPT
   ```

4. Configuration VXLAN correcte :
   ```bash
   ovs-vsctl list interface vxlan-lan | grep options
   ```

**Solution** :
- Recréer le tunnel VXLAN
- Vérifier la configuration réseau physique
- Redémarrer le service Open vSwitch :
  ```bash
  systemctl restart openvswitch-switch
  ```

#### Les deux pfSense sont en mode MASTER

**Symptômes** :
- Statut MASTER sur les deux nœuds simultanément
- Conflit d'adresses IP virtuelles

**Causes possibles** :
- Communication CARP bloquée
- Mot de passe CARP différent
- VHID Group différent

**Solutions** :

1. Vérifier les règles de pare-feu autorisant le protocole CARP
2. Vérifier l'identité du mot de passe CARP sur les deux nœuds
3. Vérifier l'identité du VHID Group
4. Tester la communication entre les pfSense :
   ```bash
   # Depuis le Master
   ping 172.16.0.2
   ```

#### La synchronisation ne fonctionne pas

**Symptômes** :
- Modifications non répliquées vers le Backup
- Messages d'erreur dans les logs système

**Vérifications** :

1. Connectivité HTTPS vers le Backup :
   ```bash
   # Depuis le Master
   telnet 172.16.0.2 443
   ```

2. Identifiants de connexion corrects dans **System** → **High Avail. Sync**

3. Règles de pare-feu autorisant le port 443

**Solution** :
- Reconfigurer la synchronisation haute disponibilité
- Vérifier les certificats SSL/TLS
- Consulter les logs dans **Status** → **System Logs** → **System**

#### Les états de connexion ne se synchronisent pas

**Symptômes** :
- Connexions interrompues lors du basculement
- États différents entre Master et Backup

**Vérifications** :

1. pfsync activé dans **System** → **High Avail. Sync**
2. Interface de synchronisation correctement configurée (LAN)
3. Règles de pare-feu autorisant le protocole pfsync

**Solution** :
- Vérifier la configuration pfsync
- Activer les logs pfsync pour diagnostic
- Redémarrer le service pfsync :
  ```bash
  /etc/rc.reload_interfaces
  ```

### 6.2 Commandes de diagnostic

#### Diagnostic Open vSwitch :

```bash
# État général
ovs-vsctl show

# État du tunnel
ovs-vsctl list interface vxlan-lan | grep -E "link_state|error|statistics"

# Flux OpenFlow
ovs-ofctl dump-flows vmbr1

# Table MAC
ovs-appctl fdb/show vmbr1
```

#### Diagnostic réseau depuis pfSense :

```bash
# Test de connectivité
ping -c 4 [IP_destination]

# Trace route
traceroute [IP_destination]

# État des interfaces
ifconfig

# États de connexion
pfctl -ss

# Tables CARP
ifconfig | grep carp

# Logs pfsync
tcpdump -i [interface_LAN] proto pfsync
```

#### Capture de trafic VXLAN :

```bash
# Sur le nœud Proxmox
tcpdump -i [interface_physique] -n port 4789 -vv
```

---

## Référence des commandes

### Commandes Open vSwitch essentielles

#### Gestion des bridges

```bash
# Lister tous les bridges
ovs-vsctl list-br

# Créer un bridge
ovs-vsctl add-br [nom_bridge]

# Supprimer un bridge
ovs-vsctl del-br [nom_bridge]

# Lister les ports d'un bridge
ovs-vsctl list-ports [nom_bridge]
```

#### Gestion des tunnels VXLAN

```bash
# Créer un tunnel VXLAN
ovs-vsctl add-port [bridge] [nom_tunnel] \
  -- set interface [nom_tunnel] type=vxlan \
     options:remote_ip=[IP_distante] \
     options:key=[clé_VXLAN]

# Supprimer un tunnel
ovs-vsctl del-port [bridge] [nom_tunnel]

# Modifier l'IP distante
ovs-vsctl set interface [nom_tunnel] options:remote_ip=[nouvelle_IP]

# Voir les détails d'un tunnel
ovs-vsctl list interface [nom_tunnel]
```

#### Diagnostic et vérification

```bash
# Configuration complète
ovs-vsctl show

# État d'une interface
ovs-vsctl list interface [nom_interface] | grep -E "link_state|error"

# Statistiques
ovs-vsctl list interface [nom_interface] | grep statistics

# Flows OpenFlow
ovs-ofctl dump-flows [bridge]

# Table MAC
ovs-appctl fdb/show [bridge]
```

### Commandes pfSense utiles

#### Configuration réseau

```bash
# Recharger la configuration réseau
/etc/rc.reload_interfaces

# Redémarrer tous les services
/etc/rc.restart_services

# État des interfaces
ifconfig

# Table de routage
netstat -rn
```

#### Diagnostic CARP

```bash
# État CARP
ifconfig | grep carp

# Forcer le mode maintenance CARP
/etc/rc.carp_maintenance_mode start

# Sortir du mode maintenance
/etc/rc.carp_maintenance_mode stop
```

#### Diagnostic pfsync

```bash
# Capturer le trafic pfsync
tcpdump -i [interface] proto pfsync

# Statistiques pfsync
pfctl -vvsync
```

---

## Ressources

### Documentation officielle

- [Documentation Proxmox VE](https://pve.proxmox.com/wiki/Main_Page)
- [Documentation pfSense](https://docs.netgate.com/pfsense/en/latest/)
- [Documentation Open vSwitch](https://docs.openvswitch.org/)
- [CARP Protocol Specification](https://www.openbsd.org/faq/pf/carp.html)

### Guides et tutoriels

- [Proxmox VE Network Configuration](https://pve.proxmox.com/wiki/Network_Configuration)
- [pfSense High Availability](https://docs.netgate.com/pfsense/en/latest/highavailability/index.html)
- [Open vSwitch VXLAN Tutorial](https://docs.openvswitch.org/en/latest/howto/vxlan/)

### Outils de diagnostic

- [Wireshark](https://www.wireshark.org/) - Analyseur de protocoles réseau
- [iperf3](https://iperf.fr/) - Outil de test de performance réseau
- [tcpdump](https://www.tcpdump.org/) - Analyseur de paquets en ligne de commande

---

## Licence

Ce guide est distribué sous licence MIT. Vous êtes libre de l'utiliser, le modifier et le distribuer selon les termes de cette licence.

---

## Contribution

Les contributions sont les bienvenues. Pour proposer des améliorations :

1. Créer une issue pour discuter des modifications proposées
2. Soumettre une pull request avec les changements détaillés
3. Respecter le format et la structure du document

---

**Version** : 1.0  
**Dernière mise à jour** : Janvier 2026  
**Auteur** : Documentation technique pfSense HA
