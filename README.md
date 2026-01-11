# Configuration d'un failover PFsense avec promox

## Prérequis

* Système d'exploitation : Promox 9.1.4
* Accès administrateur (sudo ou root)
* Cluster avec minimum 2 machines

> [!caution]
> Cette documentation a été testée et validée sur des serveurs Proxmox **9.1.4** et deux PFsense **2.8.1-RELEASE**.
> Cette documentation a été testée avec une carté réseau physique et une virtuelle OVS
> En cas de problème, vérifiez votre configuration systéme et réseau.

---
1. Installation de PFsense

1.1 Installer PFsense

- iso : https://github.com/ipsec-dev/pfsense-iso/releases
- Création de la VM sur les deux serveurs

2. Installation des packet nécessaire et ajout des carte réseaux

2.1 Installation de openvswitch-switch

```bash
apt update ; apt dist-upgrade -y

apt install openvswitch-switch -y
```

2.2 Installation des carte réseaux 

Sur les deux serveur example : Proxmox 1 et Proxmox 2
```bash
System --> Network --> Create --> OVS Bridge --> Create --> Apply Configuration
```

2.3 Ajouts des carte réseau sur les VM PFsense

Sur les deux serveur example : Proxmox 1 et Proxmox 2
```bash
Hardware --> Add --> Network Device --> Bridge : vmbr1 --> add 
```

3. Configuration de openvswitch

Création d'un tunnel vxlan sur les serveurs:

```bash
ovs-vsctl add-port vmbr1 vxlan-lan \
  -- set interface vxlan-lan type=vxlan \
     options:remote_ip=192.168.1.20 \
     options:key=2000
```

et inversement sur l'autre serveur 

```bash
ovs-vsctl add-port vmbr1 vxlan-lan \
  -- set interface vxlan-lan type=vxlan \
     options:remote_ip=192.168.1.10 \
     options:key=2000
```

Pour voir la configuation : 

```bash
ovs-vsctl show
```

Vérifier l'état de l'interface : 
```bash
ovs-vsctl list interface vxlan-lan | grep -E "link_state|error"
```

**Résultat attendu :**
```
error               : []
link_state          : up
```
