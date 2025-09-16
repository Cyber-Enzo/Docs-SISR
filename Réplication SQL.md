
# Mise en place de la réplication de base de données MariaDB

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