
# 1. Mise en place de la réplication de base de données MariaDB

## 1. Préparation du dossier de logs

Créez le dossier de logs avec les bons droits :

```bash
sudo mkdir -m 2750 /var/log/mysql
sudo chown mysql /var/log/mysql
```

---

## 2. Configuration du serveur maître

Modifiez le fichier de configuration MariaDB :

```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

Dans ce fichier :

- Commentez la ligne suivante :
	```ini
	#bind-address = 127.0.0.1
	```
- Décommentez les lignes suivantes :
	```ini
	log_error = /var/log/mysql/error.log
	server-id = 1
	log_bin = /var/log/mysql/mariadb-bin
	expire_logs_days = 10
	max_binlog_size = 100M
	```
- Ajoutez la ligne suivante pour ne répliquer qu'une base spécifique :
	```ini
	binlog_do_db = "nom_de_la_base_de_donnees"
	```

---

## 3. Redémarrage du service MariaDB

Après modification, redémarrez le service :

```bash
sudo systemctl restart mariadb
```


---

## 4. Création de l'utilisateur de réplication (sur le serveur maître)

Connectez-vous à MariaDB :

```bash
mysql
```

Créez un utilisateur dédié à la réplication :

```sql
CREATE USER 'replicateur'@'%' IDENTIFIED BY 'mot_de_passe';
GRANT REPLICATION SLAVE ON *.* TO 'replicateur'@'%';
```

Verrouillez les tables pour garantir la cohérence lors de la copie :

```sql
FLUSH TABLES WITH READ LOCK;
```

Affichez le statut du maître (notez le nom du fichier et la position) :

```sql
SHOW MASTER STATUS;
```

> Cela doit afficher un fichier `mysql-bin.00000x`, la position courante, et le Binlog_Do_DB défini précédemment.

---

## 5. Configuration du serveur esclave

1. **Préparez le dossier de logs** (comme sur le maître) :
	 ```bash
	 sudo mkdir -m 2750 /var/log/mysql
	 sudo chown mysql /var/log/mysql
	 ```

2. **Modifiez le fichier de configuration** `/etc/mysql/mariadb.conf.d/50-server.cnf` :
	 - Décommentez `log_error`, `server-id` (mettez `server-id = 2`), `max_binlog_size`.
	 - Ajoutez :
		 ```ini
		 master-retry-count = 20
		 replicate-do-db = nom_de_la_base_de_donnees
		 ```

3. **Redémarrez le service MariaDB** :
	 ```bash
	 sudo systemctl restart mariadb
	 ```

4. **Configurez la réplication** :
	 - Connectez-vous à MariaDB :
		 ```bash
		 mysql -u root -p
		 ```
	 - Arrêtez l'esclave (si déjà configuré) :
		 ```sql
		 STOP SLAVE;
		 ```
	 - Indiquez à l'esclave qui est le maître (en une seule ligne) :
		 ```sql
		 CHANGE MASTER TO master_host='172.16.0.10', master_user='replicateur', master_password='mot_de_passe', master_log_file='mysql-bin.000001', master_log_pos=328;
		 ```
		 > Remplacez l'IP, le nom d'utilisateur, le mot de passe, le fichier et la position par ceux obtenus sur le maître.

	 - Redémarrez l'esclave :
		 ```sql
		 START SLAVE;
		 ```

5. **Vérifiez le statut de la réplication** :
	 ```sql
	 SHOW SLAVE STATUS \G;
	 ```
	 > Vous devez voir l'IP du maître, le nom du user, le log file, la position, et deux lignes `Yes` pour `Slave_IO_Running` et `Slave_SQL_Running`.


---

## 6. Vérification de la réplication et tests

### 1. Débloquer les tables sur le maître

Dans MariaDB sur le serveur maître :
```sql
UNLOCK TABLES;
```

### 2. Utiliser la base de données et afficher les tables

```sql
USE gsb_valide;
SHOW TABLES;
```

### 3. Afficher le contenu d'une table

```sql
SELECT * FROM Visiteur;
```

### 4. Changer le mot de passe d'un utilisateur

```sql
UPDATE gsb_valide.Visiteur SET mdp='toto' WHERE login='agest';
```

### 5. Vérifier la réplication sur l'esclave

Sur le serveur esclave, dans MariaDB :
```sql
SELECT * FROM Visiteur;
```
Le mot de passe du login `agest` doit être passé à `toto`.

### 6. Vérifier la position des fichiers de logs

Sur le maître :
```sql
SHOW MASTER STATUS;
```
Sur l'esclave :
```sql
SHOW SLAVE STATUS \G;
```
> Remarque : Les deux machines doivent avoir la même position pour garantir la synchronisation.


# 2. — Partie 2 : Mode multi‑maître (failover)

Cette deuxième partie explique comment préparer les deux serveurs pour un scénario "multi‑maître" (chaque serveur est maître et esclave) afin de gérer la défaillance du serveur principal.

Important : avant d'appliquer ces modifications, assurez‑vous d'avoir sauvegardé les fichiers de configuration et les données en faisant une snapshot.

## Sur le maître

Éditez le fichier de configuration :

```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

Réécrivez/ajoutez les deux paramètres vus sur l'esclave et ajoutez la ligne `log-slave-updates` :

```ini
master-retry-count = 20
replicate-do-db = nom_de_la_base_de_donnees
log-slave-updates
```

`master-retry-count` augmente la tolérance aux erreurs de connexion au maître. `replicate-do-db` restreint la réplication à la base indiquée. `log-slave-updates` permet à cet hôte d'écrire dans son propre binlog les mises à jour reçues (nécessaire en configuration multi‑maître).

## Sur l'autre serveur (l'esclave)

Éditez également son fichier de configuration :

```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

Appliquez les modifications suivantes :

- Commentez la ligne :
	```ini
	#bind-address = 127.0.0.1
	```
- Décommentez/ajoutez la ligne `log-bin` si nécessaire :
	```ini
	log_bin = /var/log/mysql/mariadb-bin
	```
- Ajoutez les lignes suivantes :
	```ini
	binlog_do_db = nom_de_la_base_de_donnees
	log-slave-updates
	```

## Redémarrage des services

Après modification des fichiers de configuration sur chaque serveur :

```bash
sudo systemctl restart mariadb
```

Vérifiez ensuite le statut de réplication et faites des tests d'écriture/lire pour confirmer la synchronisation.
     

Sur le serveur esclave (servweb2) :

- Créer l'utilisateur replicateur :
	```sql
	CREATE USER 'replicateur'@'%' IDENTIFIED BY 'mot_de_passe';
	```
- Lui donner les droits :
	```sql
	GRANT REPLICATION SLAVE ON *.* TO 'replicateur'@'%';
	```


---

## Synchronisation finale et bascule multi-maître

### 1. Arrêter l'esclave sur les deux serveurs

Dans MariaDB sur chaque serveur :
```sql
STOP SLAVE;
```

### 2. Récupérer les informations de log

Sur chaque serveur, notez le nom du fichier de log binaire (`mysql-bin.00000x`) et la position (`SHOW MASTER STATUS;`).

### 3. Configurer la bascule croisée

Sur le **serveur 1** (remplacez les valeurs par celles du serveur 2) :
```sql
CHANGE MASTER TO master_host='172.16.0.11', master_user='replicateur', master_password='mot_de_passe', master_log_file='mysql-bin.000001', master_log_pos=328;
```
Sur le **serveur 2** (remplacez les valeurs par celles du serveur 1) :
```sql
CHANGE MASTER TO master_host='172.16.0.10', master_user='replicateur', master_password='mot_de_passe', master_log_file='mysql-bin.000002', master_log_pos=342;
```

### 4. Redémarrer l'esclave sur chaque serveur

```sql
START SLAVE;
```

---

## Intégration avec le cluster (Pacemaker/CRM)

Afin de pouvoir configurer le cluster SQL, il faut au préalable installer et configurer Corosync et Pacemaker sur vos deux serveurs SQL, de la même manière que pour les serveurs web.

### 1. Installation des paquets sur srv-sql1 et srv-sql2

Sur les deux serveurs, installez les paquets nécessaires :
```bash
sudo apt update
sudo apt install corosync pacemaker crmsh
```

### 2. Génération et copie de la clé d'authentification

Sur **srv-sql1 uniquement**, générez la clé `authkey` :
```bash
sudo corosync-keygen
```

Copiez cette clé vers **srv-sql2** pour que les deux nœuds puissent communiquer :
```bash
sudo scp /etc/corosync/authkey etudiant@<IP_SRV_SQL2>:/etc/corosync/authkey
```
> **Note :** Si la copie directe en root n'est pas possible, utilisez l'utilisateur `etudiant` comme détaillé dans la documentation du cluster web, en n'oubliant pas de remettre les droits (`chmod 400` et `chown root:root`) sur le fichier `/etc/corosync/authkey` d'arrivée.

### 3. Configuration de Corosync

Sur les deux nœuds, éditez le fichier de configuration `/etc/corosync/corosync.conf` (en faisant une sauvegarde au préalable) et modifiez la section `nodelist` pour l'adapter à vos serveurs SQL :

```ini
nodelist {
    node {
        name: srv-sql1
        nodeid: 1
        ring0_addr: <IP_de_srv-sql1>
    }
    node {
        name: srv-sql2
        nodeid: 2
        ring0_addr: <IP_de_srv-sql2>
    }
}
```

### 4. Démarrage des services

Sur les deux serveurs, activez et démarrez les services :
```bash
sudo systemctl enable --now corosync pacemaker
```

Vérifiez que les deux nœuds communiquent correctement et sont "Online" :
```bash
crm status
```
> Le résultat doit indiquer : `Online: [ srv-sql1 srv-sql2 ]`

---

### 5. Configuration des ressources dans Pacemaker

Une fois le cluster de base fonctionnel et les deux nœuds reconnus, vous pouvez configurer les propriétés CRM (depuis n'importe quel nœud) :
```bash
crm configure property stonith-enabled=false
crm configure property no-quorum-policy="ignore"
```

Ensuite, ajoutez la ressource du service MySQL et clonez-la pour qu'elle soit gérée sur les deux serveurs en même temps :
```bash
crm configure primitive serviceMySQL ocf:heartbeat:mysql params socket=/var/run/mysqld/mysqld.sock
crm configure clone cServiceMySQL serviceMySQL
```

Vérifiez enfin l'état complet de vos ressources et du cluster :
```bash
crm status
```

    




