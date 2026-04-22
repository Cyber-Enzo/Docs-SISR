# Configuration des Interfaces Réseau sous Linux (`/etc/network/interfaces`)

Ce guide explique comment configurer les interfaces réseau sur les distributions Linux basées sur Debian (Debian, anciennes versions d'Ubuntu Server, Proxmox, etc.) en utilisant le fichier `/etc/network/interfaces`.

## 1. Commandes de base utiles
Avant de commencer, voici quelques commandes pour diagnostiquer et appliquer vos changements :
- `ip a` : Afficher la liste des interfaces réseau et leurs adresses IP.
- `ifup <interface>` : Activer/monter une interface (ex: `ifup ens192`).
- `ifdown <interface>` : Désactiver/démonter une interface (ex: `ifdown ens192`).
- `systemctl restart networking` : Redémarrer le service réseau pour appliquer les modifications globalement.

---

## 2. Configuration SANS VLAN

Pour éditer la configuration, ouvrez le fichier avec les privilèges administrateur :
```bash
sudo nano /etc/network/interfaces
```

### A. Configuration en DHCP (IP dynamique)
L'interface obtient son adresse IP automatiquement depuis un serveur DHCP présent sur le réseau.

```text
# L'interface de bouclage (localhost) - Généralement déjà présente et requise
auto lo
iface lo inet loopback

# Configuration de l'interface ens192 en DHCP
allow-hotplug ens192
iface ens192 inet dhcp
```
*(Remarque : L'instruction `allow-hotplug ens192` permet à l'interface de s'activer automatiquement au démarrage du système).*

### B. Configuration en IP Statique
L'interface est configurée avec une adresse IP fixe. C'est l'usage le plus courant pour un serveur.

```bash
# Configuration de l'interface ens192 en IP Statique
allow-hotplug ens192
iface ens192 inet static
    address 192.168.1.10
    netmask 255.255.255.0
    gateway 192.168.1.254
    # Serveurs DNS (optionnel, nécessite parfois le paquet resolvconf)
    dns-nameservers 8.8.8.8 1.1.1.1
```
> **Note :** Sur les versions récentes, vous pouvez utiliser la notation CIDR directement, ce qui évite de préciser le masque : 
> `address 192.168.1.10/24`

---

## 3. Configuration AVEC VLAN (802.1Q)

Pour utiliser des VLANs, le trafic réseau doit être "taggé" (norme 802.1Q). Le système doit donc prendre en charge ce protocole.

### A. Prérequis
Assurez-vous que le paquet `vlan` est installé et que le module noyau est chargé :
```bash
# Installation du paquet
sudo apt update
sudo apt install vlan

# Chargement du module kernel 802.1Q
sudo modprobe 8021q

# Pour s'assurer que le module soit chargé à chaque redémarrage :
echo "8021q" | sudo tee -a /etc/modules
```

### B. Configuration dans le fichier
Il faut configurer l'interface physique (qui fera office de port "Trunk") et les sous-interfaces virtuelles liées aux VLANs (qui porteront le tag VLAN).

Dans cet exemple, l'interface physique est `ens192`. Nous allons lui attribuer le **VLAN 10** et le **VLAN 20**.

```bash
# 1. Configuration du VLAN 10 (ex: réseau Admin)

allow-hotplug ens192.10
iface ens192.10 inet static
    address 10.10.10.100
    netmask 255.255.255.0
    gateway 10.10.10.254
    vlan-raw-device ens192
    dns-nameservers 8.8.8.8

# 2. Configuration du VLAN 20 (ex: réseau Utilisateurs)

allow-hotplug ens192.20
iface ens192.20 inet static
    address 10.20.20.100
    netmask 255.255.255.0
    vlan-raw-device ens192
    dns-nameservers 8.8.8.8
```

> **Astuce :** 
> - La notation `ens192.10` indique au système qu'il s'agit du VLAN 10 sur l'interface matérielle `ens192`.
> - La directive `vlan-raw-device ens192` précise l'interface physique parente. Elle est recommandée bien qu'elle soit souvent devinée automatiquement grâce au nommage de l'interface.

### C. Appliquer les modifications
Après avoir sauvegardé et quitté le fichier (`Ctrl+X`, puis `Y` et `Entrée`), appliquez les changements.

Le plus propre pour les VLANs est de monter les interfaces manuellement ou de relancer le service :
```bash
# Activer le Trunk
sudo ifup ens192

# Activer les sous-interfaces VLAN
sudo ifup ens192.10
sudo ifup ens192.20

# Ou redémarrer l'ensemble du service réseau
sudo systemctl restart networking
```

Pour vérifier que tout fonctionne, utilisez la commande `ip a` ou `ip addr`. 
Vous devriez y voir des interfaces nommées `ens192.10@ens192` et `ens192.20@ens192` avec leurs adresses IP respectives.
