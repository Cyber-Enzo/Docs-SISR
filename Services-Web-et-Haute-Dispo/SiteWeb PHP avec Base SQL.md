# Configuration d'un Site Web PHP avec Base SQL

## 1. Installation des prérequis

**Sur le serveur Web :**
Il faut installer PHP, le module PHP pour MySQL et le client MariaDB.
```bash
sudo apt update
sudo apt install php php-mysql mariadb-client
```

**Sur le serveur SQL :**
Il faut installer le serveur MariaDB.
```bash
sudo apt update
sudo apt install mariadb-server
```

## 2. Configuration de la base de données (sur le serveur SQL)

Connectez-vous à MySQL :
```bash
mysql -u root -p
```

Dans l'invite de commande MySQL, exécutez les requêtes suivantes. 
*Note : Veillez à créer l'utilisateur qui correspond à celui défini dans les fichiers web de votre projet (bon nom d'utilisateur, bon mot de passe). Utilisez l'adresse IP du serveur Web à la place de `localhost` pour autoriser la connexion distante.*

```sql
-- Créer la base de données
CREATE DATABASE nom_de_la_base;

-- Créer l'utilisateur avec son mot de passe et l'autoriser depuis l'IP du serveur Web
CREATE USER 'nom_utilisateur'@'IP_SERVEUR_WEB' IDENTIFIED BY 'mot_de_passe';

-- Assigner les droits à l'utilisateur sur la base de données
GRANT ALL PRIVILEGES ON nom_de_la_base.* TO 'nom_utilisateur'@'IP_SERVEUR_WEB';
FLUSH PRIVILEGES;

-- Voir les utilisateurs créés 
SELECT User, Host FROM mysql.user;

-- Quitter MySQL
exit
```

## 3. Importation des données

Transférez vos fichiers `.sql` sur le serveur SQL (par exemple dans `/home/etudiant`), puis importez-les dans la base de données avec la commande suivante :

```bash
mysql -u root -p nom_de_la_base < nom_du_fichier.sql
```

## 4. Vérification du contenu de la base de données

Reconnectez-vous à MySQL pour bien vérifier le contenu de la base de données :
```bash
mysql -u root -p
```

```sql
USE nom_de_la_base;
SHOW TABLES;
SELECT * FROM nom_de_la_table;
exit
```

## 5. Configuration de l'accès distant à MariaDB

Si ce n'est pas déjà fait, il faut autoriser MariaDB à écouter sur d'autres interfaces que localhost.

Éditez le fichier de configuration de MariaDB sur le serveur SQL :
```bash
sudo nano /etc/mysql/mariadb.conf.d/50-server.cnf
```

Trouvez la ligne `bind-address` et modifiez-la pour écouter sur toutes les interfaces :
```ini
bind-address = 0.0.0.0
```

## 6. Redémarrage des services

**Sur le serveur SQL :**
Redémarrez le service MariaDB pour appliquer les modifications de configuration :
```bash
sudo systemctl restart mariadb.service
```

**Sur le serveur Web :**
Pour terminer, allez sur le serveur Web et redémarrez le service Apache2 pour qu'il prenne en compte les nouveaux modules :
```bash
sudo systemctl restart apache2
```
