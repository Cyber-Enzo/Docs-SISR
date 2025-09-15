# Mise en oeuvre d'un Cluster Web avec Corosync

Ce guide explique étape par étape comment installer et configurer un cluster web haute disponibilité avec Corosync, Pacemaker et crmsh.

---

## 1. Installation des paquets nécessaires

Installez Corosync, Pacemaker et crmsh sur chaque nœud du cluster :

```bash
apt install corosync pacemaker crmsh
```

---

## 2. Génération de la clé d'authentification

Générez la clé d'authentification pour Corosync :

```bash
corosync-keygen
```

Vérifiez la présence du fichier `authkey` :

```bash
ls /etc/corosync
```

Vous devez voir le fichier `authkey`.

---

## 3. Sauvegarde de la configuration

Avant toute modification, sauvegardez le fichier de configuration :

```bash
cp /etc/corosync/corosync.conf /etc/corosync/corosync.conf.save
```

---

## 4. Configuration du cluster

Modifiez le fichier `/etc/corosync/corosync.conf` pour définir les paramètres du cluster et les nœuds :

```ini
# /etc/corosync/corosync.conf

totem {
    version: 2
    cluster_name: cluster_web
    crypto_cipher: aes256
    crypto_hash: sha1
    clear_node_high_bit: yes
}

logging {
    fileline: off
    to_logfile: yes
    logfile: /var/log/corosync/corosync.log
    to_syslog: no
    debug: off
    timestamp: on
    logger_subsys {
        subsys: QUORUM
        debug: off
    }
}

quorum {
    provider: corosync_votequorum
    expected_votes: 2
    two_nodes: 1
}

nodelist {
    node {
        name: serv1   # À modifier selon le nom du serveur
        nodeid: 1
        ring0_addr: 172.16.0.10
    }
    node {
        name: serv2   # À modifier selon le nom du serveur
        nodeid: 2
        ring0_addr: 172.16.0.11
    }
}

service {
    ver: 0
    name: pacemaker
}
```

- Adaptez les noms (`serv1`, `serv2`) et les adresses IP à votre infrastructure.

---

## 5. Tester la configuration


Après avoir démarré Corosync, vérifiez la configuration du cluster et la détection des nœuds avec :

```bash
corosync-cfgtool -s
```

Cette commande affiche l'IP et les deux nœuds du cluster. Vérifiez que les deux serveurs apparaissent bien dans la sortie.

---

## 6. Clonage du serveur web

Pour assurer la haute disponibilité, éteignez le serveur web principal et effectuez un clonage complet sur le second nœud.

---

Ce fichier .markdown explique chaque étape et affiche tous les blocs de configuration et commandes nécessaires pour mettre en œuvre un cluster web avec Corosync.
