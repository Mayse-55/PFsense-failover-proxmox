# Configuration d'un failover PFsense avec promox

## Prérequis

* Système d'exploitation : Promox 9.1.4
* Accès administrateur (sudo ou root)
* Cluster avec minimum 2 machines

> [!caution]
> Cette documentation a été testée et validée sur des serveurs Proxmox **9.1.4** et deux PFsense **2.8.1-RELEASE**  .  
> En cas de problème, vérifiez votre configuration systéme et réseau.

> **Documentation Master Server :**  
> Pour l'installation du serveur Master, consultez : [https://github.com/Mayse-55/MooseFS-Master](https://github.com/Mayse-55/MooseFS-Master)

Ajout des carte réseau 
