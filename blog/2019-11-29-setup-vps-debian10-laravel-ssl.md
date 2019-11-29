---
title: Setup VPS sous Débian 10 (Buster) pour Laravel
path: /setup-vps-debian10-laravel-ssl
date: 2019-11-28
summary: Configurer de A à Z un VPS sous débian 1O pour héberger un site Laravel avec Apache2 en SSL.
tags: ['Laravel', 'Administration système', 'Debian 10', 'Apache2','npm']
---
# Setup VPS sous Débian 10 (Buster) pour Laravel
> Commencez par vous connecter à votre VPS en ssh.
> Passez en root avec la commande `su -`.
> Le `-` est important car il indique au shell d'utiliser les variables d'environnement du root et non celles de votre utilisateur de départ ...

Nous aurons besoin d'installer les outils suivants:
* curl
* wget
* git
* outils de compilation de base

```bash
 apt-get update && apt-get install curl git gcc g++ make wget
```
# Installation de NodeJS pour npm

```bash
curl -sL https://deb.nodesource.com/setup_12.x -o nodesource_setup.sh
bash nodesource_setup.sh
apt-get install -y nodejs
```


#Installation du Serveur Apache et de PHP
```bash
 apt-get install apache2
```
## Activation des modules apache
Le routage de Laravel a besoin du rewriting d'url pour fonctionner correctement.
```bash
a2enmod rewrite
```

## Gestion des droits de fichiers utilisateur - Apache2
C'est une bonne idée d'ajouter votre utilisateur au groupe www-data et de lui donner l'accés aux vos fichiers:

Imaginons que l'utilisateur s'appelle 'utilisateur' et que les fichiers du site se trouvent dans le répertoire /var/www/domaine.com.

```bash
usermod -aG utilisateur www-data
chown -R utilisateur:www-data /var/www/domaine.com
chmod -R 755 /var/www/domaine.com
```



## Mise en place de PHP
Pour fonctionner correctement, Laravel a besoin au moins de la version 7.2 de PHP.

```bash
 apt-get install php7.3-common php7.3-cli  php7.3-gd php7.3-mysql php7.3-curl php7.3-intl php7.3-mbstring php7.3-bcmath php7.3-imap php7.3-xml php7.3-zip
```
### Installation de composer manager
Composer est disponnible directement dans les dépôts de Débian Buster.
```bash
apt-get install composer
```

# Installation de Yarn package manager

```bash
curl -sL https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add -
echo "deb https://dl.yarnpkg.com/debian/ stable main" | tee /etc/apt/sources.list.d/yarn.list
apt-get update && apt-get install yarn
```

# Configuration des vhosts
Créer un virtual host dans Apache2 est très simple:
* créer un fichier /etc/apache2/site-aviable/*.conf
* activer le site avec la commande `a2ensite *.conf`

Un lien symbolique sera créer dans /etc/apache2/site-enable pour indiquer que le site est activé. Il ne reste plus qu'a reloder apache avec la commande `systemctrl restart apache2`.

Example de fichier de configuration:

```bash
<VirtualHost *:80>
    ServerName domaine.com

    ServerAdmin contact@domaine.com
    DocumentRoot /var/www/domaine.com/public

    # On redirige toutes les requettes en HTTPS
    Redirect / https://www.domaine.com

    ErrorLog ${APACHE_LOG_DIR}/domaine.com.error.log
    CustomLog ${APACHE_LOG_DIR}/domaine.com.access.log combined
</VirtualHost>

<VirtualHost _default_:443>
    # Configuration du domaine et de l'alias
    ServerName domaine.com
    ServerAlias www.domaine.com

    ServerAdmin contact@domaine.com

    # Répertoire de base du site Web
    DocumentRoot /var/www/domaine.com/public

    # On authorise l'utilisation des .htaccess pour Laravel, important !
    <Directory /var/www/domaine.com/>
             AllowOverride All
    </Directory>

    # Configuration SSL
    SSLEngine On
    SSLCertificateFile /etc/ssl/certs/_.domaine.com.crt
    SSLCertificateKeyFile /etc/ssl/private/_.domaine.com.key
    SSLCertificateChainFile /etc/ssl/certs/GandiStandardSSLCA2.pem

    # Choix des fichiers de log
    ErrorLog ${APACHE_LOG_DIR}/domaine.com.error.log
    CustomLog ${APACHE_LOG_DIR}/domaine.com.access.log combined
</VirtualHost>
```