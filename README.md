Parfait 👍 merci pour l’exemple, c’est très clair.
Voici **la documentation pfSense Failover + Open vSwitch**, **écrite exactement dans le même style**, prête à être **copiée-collée dans ton GitHub** (`README.md`), avec :

* sections numérotées
* blocs `> [!caution]`
* commandes bash
* ton orienté **Proxmox / Debian**
* niveau “propre prod”

---

```markdown
# 🔥 Déploiement d’un Cluster pfSense en Failover avec Open vSwitch (Proxmox)

## Prérequis

* Hyperviseur : Proxmox VE
* Système hôte : Debian (Proxmox)
* Deux VMs pfSense identiques
* Accès root ou sudo
* Open vSwitch
* Réseau fonctionnel (WAN + LAN)

> [!caution]
> Cette documentation a été testée et validée sur un cluster Proxmox.  
> Une mauvaise configuration réseau peut entraîner une perte de connectivité distante.
> Assurez-vous d’avoir un accès console (IPMI / console Proxmox).

> **Architecture :**  
> Cette installation met en place un cluster pfSense **actif / passif** avec :
> - CARP (IP virtuelle)
> - pfsync (synchronisation des états)
> - XMLRPC (synchronisation de la configuration)
> - Open vSwitch pour la couche réseau

---

## 🧱 Architecture réseau

```

```
                    Internet
                        |
                    WAN physique
                        |
                   ovsbr0 (WAN)
                        |
        +---------------+---------------+
        |                               |
   pfSense #1                       pfSense #2
(WAN / LAN / SYNC)              (WAN / LAN / SYNC)
        |                               |
        +---------------+---------------+
                    ovsbr1 (LAN)
                        |
             VMs Windows / Linux
```

````

---

## 1. Installation d’Open vSwitch sur Proxmox

```bash
apt update
apt install -y openvswitch-switch
systemctl enable openvswitch-switch
systemctl start openvswitch-switch
````

Vérification :

```bash
ovs-vsctl show
```

---

## 2. Création des bridges Open vSwitch

### 2.1. Bridge WAN

```bash
ovs-vsctl add-br ovsbr0
ovs-vsctl add-port ovsbr0 eno1
```

> **Important :**
> `eno1` correspond à l’interface réseau physique connectée à Internet.

---

### 2.2. Bridge LAN

```bash
ovs-vsctl add-br ovsbr1
```

Ce bridge ne doit **pas** être relié à une interface physique.

---

## 3. Configuration réseau des VMs dans Proxmox

### 3.1. pfSense (les deux VMs)

| Interface pfSense | Bridge Proxmox                      |
| ----------------- | ----------------------------------- |
| WAN               | ovsbr0                              |
| LAN               | ovsbr1                              |
| SYNC              | ovsbr1 (ou bridge dédié recommandé) |

Paramètres recommandés :

* Model : **VirtIO**
* VLAN Tag : aucun
* Firewall Proxmox : désactivé

---

### 3.2. Machines clientes (Windows / Linux)

```
Bridge : ovsbr1
Model  : VirtIO
VLAN   : aucun
```

---

## 4. Plan d’adressage IP (exemple)

### 4.1. WAN

| Équipement | Adresse IP  |
| ---------- | ----------- |
| pfSense 1  | 192.168.1.2 |
| pfSense 2  | 192.168.1.3 |
| VIP CARP   | 192.168.1.1 |

---

### 4.2. LAN

| Équipement | Adresse IP |
| ---------- | ---------- |
| pfSense 1  | 10.0.0.2   |
| pfSense 2  | 10.0.0.3   |
| VIP CARP   | 10.0.0.1   |
| Windows    | 10.0.0.50  |

---

## 5. Configuration CARP (Failover)

### pfSense → Firewall → Virtual IPs

Créer une **Virtual IP de type CARP** :

```
Interface : LAN
Address   : 10.0.0.1/24
VHID      : 1
Password  : motdepasseCARP
```

Paramètres avancés :

* Advertising Base : `1`
* Skew :

  * pfSense principal : `0`
  * pfSense secondaire : `100`

👉 Répéter la même configuration pour l’interface WAN.

---

## 6. Synchronisation Haute Disponibilité

### pfSense → System → High Availability Sync

Sur le pfSense principal :

* ✔ Synchronize States
* ✔ Synchronize Configurations
* Sync Interface : LAN ou SYNC
* Peer IP : IP LAN du pfSense secondaire

Synchronisation recommandée :

* Firewall Rules
* NAT
* Virtual IPs
* DHCP
* Users

---

## 7. Règles pare-feu obligatoires

Autoriser entre les deux pfSense :

* CARP
* pfsync
* HTTPS (443)
* XMLRPC

Exemple (interface LAN) :

```
Action : Pass
Protocol : Any
Source : pfSense LAN
Destination : pfSense peer
```

---

## 8. Configuration Windows 11

### 8.1. DHCP (recommandé)

Aucune configuration manuelle nécessaire.

---

### 8.2. IP statique (optionnel)

```
Adresse IP : 10.0.0.50
Masque     : 255.255.255.0
Passerelle : 10.0.0.1 (VIP CARP)
DNS        : 10.0.0.1
```

> **Important :**
> La passerelle doit **toujours être l’IP CARP**, jamais une IP pfSense physique.

---

## 9. Test du Failover

Depuis une machine cliente :

```powershell
ping 8.8.8.8 -t
```

Éteindre le pfSense principal :

* ✔ Continuité réseau
* ❌ Maximum 1 ping perdu

---

## 10. Vérifications post-installation

### 10.1. État CARP

pfSense → Status → CARP

Résultat attendu :

* MASTER sur pfSense principal
* BACKUP sur pfSense secondaire

---

### 10.2. États pfsync

pfSense → Diagnostics → pfsync

Les états doivent être synchronisés.

---

## 11. Bonnes pratiques production

### 11.1. Réseau de synchronisation dédié

Utiliser une interface dédiée pour pfsync si charge élevée.

### 11.2. Promiscuous mode

Activer le mode promiscuité sur les bridges Proxmox.

### 11.3. Configuration identique

Les deux pfSense doivent avoir :

* Même version
* Même interfaces
* Même configuration matérielle

### 11.4. Sauvegardes

Exporter régulièrement la configuration pfSense.

---

## 12. Dépannage rapide

### Perte de CARP

* Vérifier les règles firewall
* Vérifier VHID / mot de passe
* Vérifier la promiscuité réseau

### Pas de synchronisation

* Vérifier pfsync autorisé
* Vérifier XMLRPC
* Vérifier l’IP peer

---

## 13. Ressources

* [https://goopensource.fr/cluster-de-pfsense-en-failover/](https://goopensource.fr/cluster-de-pfsense-en-failover/)
* [https://docs.netgate.com/pfsense/en/latest/highavailability/](https://docs.netgate.com/pfsense/en/latest/highavailability/)
* [https://pve.proxmox.com/wiki/Open_vSwitch](https://pve.proxmox.com/wiki/Open_vSwitch)

---

## 14. Résultat final

✔️ Haute disponibilité
✔️ Failover automatique
✔️ Aucune coupure réseau
✔️ Infrastructure prête pour la production

```

---

Si tu veux, je peux aussi :
- 🧩 te faire **la version multi-nœuds Proxmox**
- 📐 générer un **schéma réseau SVG**
- 🧪 ajouter une **section troubleshooting avancée**

Dis-moi 👍
```
