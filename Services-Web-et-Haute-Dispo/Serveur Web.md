# Installation et Configuration d’un Serveur Web Apache sur Debian 12

## 1. Installation d’Apache

Mettez à jour l’index des paquets et installez Apache :

```bash
sudo apt update
sudo apt install apache2
```

## 2. Vérification du Serveur Web

Vérifiez que le service Apache fonctionne :

```bash
sudo systemctl status apache2
```

Pour tester, récupérez l’adresse IP du serveur :

```bash
ip a
```

Accédez à cette adresse IP dans votre navigateur pour voir la page par défaut d’Apache.

## 3. Gestion du Service Apache

- **Arrêter Apache** :  
  `sudo systemctl stop apache2`
- **Démarrer Apache** :  
  `sudo systemctl start apache2`
- **Redémarrer Apache** :  
  `sudo systemctl restart apache2`
- **Recharger la configuration** :  
  `sudo systemctl reload apache2`
- **Désactiver le démarrage automatique** :  
  `sudo systemctl disable apache2`
- **Activer le démarrage automatique** :  
  `sudo systemctl enable apache2`

## 4. Configuration des Hôtes Virtuels

### a. Créer le répertoire du site

```bash
sudo mkdir -p /var/www/votre_domaine
sudo chown -R www-data:www-data /var/www/votre_domaine
sudo chmod -R 755 /var/www/votre_domaine
```

### b. Créer une page d’accueil (Test)

```bash
sudo nano /var/www/votre_domaine/index.html
```

Ajoutez :

```html
<html>
<head>
<title>Bienvenue !</title>
</head>
<body>
<h1>Succès ! L’hôte virtuel fonctionne !</h1>
</body>
</html>
```

### c. Déployer un site existant depuis une archive ZIP (ex: appliFrais)

Si vous possédez déjà les fichiers de votre site web sous format `.zip`, vous pouvez remplacer la page de test par votre application :

1. **Autoriser le transfert de fichiers** :
   Donnez les permissions sur le dossier `/home/etudiant` pour pouvoir y déposer votre fichier avec MobaXterm :
   ```bash
   sudo chmod 777 /home/etudiant
   ```

2. **Transférer l'archive** :
   Utilisez MobaXterm pour glisser-déposer votre fichier (ex: `appliFrais.zip`) dans le dossier `/home/etudiant`.

3. **Déplacer et extraire l'archive** :
   Installez l'utilitaire `unzip`, déplacez l'archive dans `/var/www/` et dézippez-la :
   ```bash
   sudo apt install unzip
   sudo mv /home/etudiant/votre_archive.zip /var/www/
   cd /var/www/
   sudo unzip votre_archive.zip
   ```

4. **Placer les fichiers et nettoyer** :
   L'extraction a créé un dossier `files`. Déplacez son contenu vers votre dossier web (ex: `appliFrais` ou `votre_domaine`), puis supprimez l'archive et le dossier `files` devenu vide :
   ```bash
   # Déplacer le contenu de "files" vers votre domaine
   sudo mv /var/www/files/* /var/www/votre_domaine/
   
   # Supprimer le zip et le dossier temporaire
   sudo rm -rf /var/www/votre_archive.zip /var/www/files
   ```

5. **Rétablir les bonnes permissions** :
   Afin qu'Apache puisse lire vos nouveaux fichiers, réappliquez les droits :
   ```bash
   sudo chown -R www-data:www-data /var/www/votre_domaine
   sudo chmod -R 755 /var/www/votre_domaine
   ```

### d. Créer le fichier de configuration de l’hôte virtuel

```bash
sudo nano /etc/apache2/sites-available/votre_domaine.conf
```

Ajoutez :

```apache
<VirtualHost *:80>
    ServerAdmin admin@votre_domaine
    ServerName votre_domaine
    ServerAlias www.votre_domaine
    DocumentRoot /var/www/votre_domaine
    DirectoryIndex index.html index.php
    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

*(Note : Assurez-vous d'avoir `index.php` dans la ligne `DirectoryIndex` si votre site l'utilise, comme c'est souvent le cas pour les projets PHP).*

### e. Activer le site et désactiver le site par défaut

```bash
sudo a2ensite votre_domaine.conf
sudo a2dissite 000-default.conf
```

### f. Vérifier la configuration

```bash
sudo apache2ctl configtest
```

Vous devriez voir : `Syntax OK`.

### g. Redémarrer Apache

```bash
sudo systemctl restart apache2
```

---

Votre serveur Apache sur Debian 12 est maintenant prêt à héberger votre site avec un hôte virtuel personnalisé.