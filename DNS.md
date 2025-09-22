# Installation d'un DNS avec redondance sous Debian 12

Ce guide explique comment configurer un serveur DNS redondant sous Debian 12 en utilisant BIND9.

---

## Prérequis

- Deux machines Debian 12 (srv-dns1 et srv-dns2)
- Accès root ou sudo
- Réseau configuré (exemple : 172.16.0.0/24)

---

## Étape 1 : Préparation des serveurs

### 1.1 Modifier la configuration réseau et renommer les machines

Sur chaque machine, modifiez le nom d’hôte et le fichier `/etc/hosts` :

```bash
sudo hostnamectl set-hostname srv-dns1   # Sur la première machine
sudo hostnamectl set-hostname srv-dns2   # Sur la seconde machine
sudo nano /etc/hosts                     # Ajoutez les IP et noms des deux serveurs
sudo reboot
```

### 1.2 Installer BIND9

Sur chaque serveur :

```bash
sudo apt update
sudo apt install bind9
```

---

## Étape 2 : Configuration du serveur maître (srv-dns1)

### 2.1 Définir la zone DNS

Éditez `/etc/bind/named.conf.local` :

```bash
zone "sodecaf.fr" {
    type master;
    file "db.sodecaf.fr";
    allow-transfer {172.16.0.4;}; # IP du serveur esclave
};

zone "0.16.172.in-addr.arpa" {
    type master;
    file "db.172.16.0.rev";
    allow-transfer {172.16.0.4;};
};
```

### 2.2 Créer les fichiers de zone

Dans `/var/cache/bind/`, créez `db.sodecaf.fr` :

```bash
sudo nano /var/cache/bind/db.sodecaf.fr
```

Contenu exemple :

```bash
$TTL 86400
@   IN  SOA srv-dns1.sodecaf.fr. hostmaster.sodecaf.fr. (
        2025092201 ; serial
        86400      ; refresh
        21600      ; retry
        3600000    ; expire
        3600 )     ; negative cache TTL
@       IN  NS  srv-dns1.sodecaf.fr.
@       IN  NS  srv-dns2.sodecaf.fr.

srv-dns1    IN  A   172.16.0.3
srv-dns2    IN  A   172.16.0.4
srv-web1    IN  A   172.16.0.10
srv-web2    IN  A   172.16.0.11
www         IN  A   172.16.0.12
web1        IN  CNAME srv-web1.sodecaf.fr.
web2        IN  CNAME srv-web2.sodecaf.fr.
```

Vérifiez la configuration avec :

```bash
sudo named-checkzone sodecaf.fr /var/cache/bind/db.sodecaf.fr
```

### 2.3 Créer la zone inverse

Copiez le fichier de zone directe et modifiez-le :

```bash
sudo cp /var/cache/bind/db.sodecaf.fr /var/cache/bind/db.172.16.0.rev
sudo nano /var/cache/bind/db.172.16.0.rev
```

Gardez la partie SOA et NS, puis ajoutez :

```bash
3   IN  PTR srv-dns1.sodecaf.fr.
4   IN  PTR srv-dns2.sodecaf.fr.
10  IN  PTR srv-web1.sodecaf.fr.
11  IN  PTR srv-web2.sodecaf.fr.
12  IN  PTR www.sodecaf.fr.
```

Vérifiez la configuration avec :

```bash
sudo named-checkzone 0.16.172.in-addr-arpa /var/cache/bind/db.172.16.0.rev
```

---

## Étape 3 : Configuration du serveur esclave (srv-dns2)

### 3.1 Définir les zones esclaves

Dans `/etc/bind/named.conf.local` :

```bash
zone "sodecaf.fr" {
    type slave;
    file "slave/db.sodecaf.fr";
    masters {172.16.0.3;};
};

zone "0.16.172.in-addr.arpa" {
    type slave;
    file "slave/db.172.16.0.rev";
    masters {172.16.0.3;};
};
```

### 3.2 Préparer le dossier slave

```bash
sudo mkdir /var/cache/bind/slave
sudo chgrp bind /var/cache/bind/slave
sudo chmod g+w /var/cache/bind/slave
```

---

## Étape 4 : Configuration des options globales

Sur les deux serveurs, éditez `/etc/bind/named.conf.options` :

- Décommentez et adaptez les forwarders pour l’accès Internet :

```bash
forwarders {
    8.8.8.8;
};
```

- Pour autoriser les requêtes de tous les réseaux :

```bash
allow-query { any; };
```

---

## Étape 5 : Redémarrage et vérification

Redémarrez BIND9 sur chaque serveur :

```bash
sudo systemctl restart bind9
```

Vérifiez la configuration :

```bash
sudo named-checkzone sodecaf.fr /var/cache/bind/db.sodecaf.fr
sudo named-checkzone 0.16.172.in-addr-arpa /var/cache/bind/db.172.16.0.rev
```

---

## Étape 6 : Tests

- Depuis Windows :

```powershell
nslookup web1.sodecaf.fr 172.16.0.3
nslookup srv-web2.sodecaf.fr 172.16.0.4
nslookup 172.16.0.10 172.16.0.3
```

- Depuis Linux :

```bash
dig web1.sodecaf.fr @172.16.0.3
dig -x 172.16.0.10 @172.16.0.3
```

---

**Votre DNS redondant est maintenant opérationnel sous Debian 12 !**

