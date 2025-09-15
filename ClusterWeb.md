# Mise en oeuvre d'un Cluster Web

Installer corosync, pacemaker et crmsh

apt install corosync pacemaker crmsh

faire corosyn-keygen et vérifier ls /etc/corosync "authkey"


sauvegarder le fichier corosync.conf dans corosync.conf.save 

modifier le nom des noeuds dans le dossier de configuration de corosync dans /etc/corosync/corosync.conf avec ça : 
```
totem {
    version: 2
    cluster_name: cluster_web
    crypto_cipher: aes256
    crypto_hash: sha1
    clear_node_high_bit:yes
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
    name: serv1 (à modifier)
    nodeid: 1
    ring0_addr: 172.16.0.10
}
    node {
    name: serv2 (à modifier)
    nodeid: 2
    ring0_addr: 172.16.0.11
}

}
    service {
    ver: 0
    name: pacemaker
}
```

Puis tester la configuration avec "corosync"

éteindre le serv web et faire un full clone du serv web 

modifier son nom dans '/etc/hosts' et dans '/etc/hostname' puis changer son ip dans '/etc/network/interfaces'

relancer ses 2 machines et faire "crm status", cela devrait afficher le nom des 2 serveurs en Online


