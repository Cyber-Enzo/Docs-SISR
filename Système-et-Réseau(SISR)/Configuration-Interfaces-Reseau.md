# Configuration d'une interface réseau (`/etc/network/interfaces`)

Ce document présente un aide-mémoire (mini-fichier de configuration) pour la gestion des interfaces réseaux sur les systèmes Debian/Ubuntu historiques (ou tout système GNU/Linux utilisant `ifupdown` et `/etc/network/interfaces`).

## 1. Sans VLAN (Configuration standard)

Ouvrez le fichier de configuration avec un éditeur de texte (ex: `nano /etc/network/interfaces`).

### IP Statique
L'interface (ici `eth0` ou `ens33`) aura une adresse fixe.

```bash
# L'interface loopback (obligatoire)
auto lo
iface lo inet loopback

# Interface réseau principale en IP statique
auto eth0
iface eth0 inet static
    address 192.168.1.50
    netmask 255.255.255.0
    gateway 192.168.1.254
    # Optionnel: Serveurs DNS
    dns-nameservers 8.8.8.8 1.1.1.1
```

### DHCP (IP Dynamique)
Si vous souhaitez que l'interface reçoive automatiquement une adresse de votre box ou de votre serveur DHCP.

```bash
auto eth0
iface eth0 inet dhcp
```

---

## 2. Avec VLAN (Type 802.1q)

Lorsqu'un port du switch est configuré en mode **TRUNK**, la machine Linux doit être capable d'écouter et de taguer les paquets sur plusieurs VLANs (sous-interfaces).

*Prérequis : Assurez-vous que le paquet `vlan` est installé (`apt install vlan`) et que le module est actif (`modprobe 8021q`).*

Ici, on garde `eth0` allumée mais sans IP (mode manuel), et on crée des "sous-interfaces" du type `nom-interface.id-vlan`.

```bash
# Interface loopback
auto lo
iface lo inet loopback

# Activation de l'interface physique (sans lui attribuer d'IP réseau)
auto eth0
iface eth0 inet manual

# --- Configuration du VLAN 10 ---
auto eth0.10
iface eth0.10 inet static
    address 192.168.10.50
    netmask 255.255.255.0
    gateway 192.168.10.254
    vlan-raw-device eth0

# --- Configuration du VLAN 20 ---
auto eth0.20
iface eth0.20 inet static
    address 192.168.20.50
    netmask 255.255.255.0
    vlan-raw-device eth0
```

---

## 3. Nettoyer et réinitialiser les interfaces (Troubleshooting)

En cas de problème (interface bloquée, ancienne adresse IP qui persiste), exécutez ces commandes dans l'ordre pour forcer la réinitialisation et repartir au propre. *(Exemple avec `ens192` et un VLAN `200`)* :

**1. Supprimer l'interface fantôme (VLAN)**
```bash
sudo ip link delete ens192.200
```
*(Note : Ne vous inquiétez pas si le terminal affiche "not found", cette commande sert juste à s'assurer que l'interface virtuelle n'existe plus).*

**2. Vider la configuration de l'interface physique**
```bash
sudo ip addr flush dev ens192
```

**3. Relancer le service réseau proprement**
```bash
sudo systemctl restart networking
```

---

## 4. Appliquer les modifications

Une fois le fichier `/etc/network/interfaces` sauvegardé (`CTRL+O`, `Entrée`, `CTRL+X` avec `nano`), il faut redémarrer le service réseau ou relancer les interfaces.

**Méthode 1 : Redémarrer le service complet**
```bash
sudo systemctl restart networking
```

**Méthode 2 : Relancer l'interface spécifiquement (plus sûr pour ne pas couper d'autres connexions)**
```bash
sudo ifdown eth0 && sudo ifup eth0

# Pour les VLANs :
sudo ifdown eth0.10 && sudo ifup eth0.10
```
