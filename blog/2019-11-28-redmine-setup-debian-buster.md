---
title: Installer Redmine avec MariaDB sur Debian 10 Buster
path: /redmine-setup-debian-buster
date: 2019-11-28
summary: Mise en place de Redmine, setup pour un environnement de production en SSL et d'intégration avec Apache2.
tags: ['Redmine', 'Gestion de projet', 'Debian 10']
---
### Mettre à jour le système

Pour commencer, assurez-vous que vos packages système sont à jour.

```bash
apt update
```

### Installer les Build tools requis et les dépendances

Pour installer Redmine à partir du code source, vous devez installer les Build tools et les dépendances requises.

```bash
apt install build-essential ruby-dev libxslt1-dev libmariadb-dev libxml2-dev zlib1g-dev imagemagick libmagickwand-dev curl vim sudo
```

Install Apache HTTP Server on Debian 10 Buster

Redmine est une application Web et vous devez donc installer un serveur Web pour y accéder.

```bash
apt install apache2
```

Ensuite, installez les modules Apache pour Passenger, le serveur Web léger pour Ruby.

```bash
apt install libapache2-mod-passenger
```

La commande ci-dessus installera également d'autres dépendances requises, y compris Ruby.

```bash
ruby -v

ruby 2.5.5p157 (2019-03-15 revision 67260) [x86_64-linux-gnu]
```

Démarrez et activez Apache pour qu'il s'exécute au démarrage du système.

```bash
systemctl enable --now apache2
```

### Créer un utilisateur du système Redmine

Lors de l'installation des dépendances de Ruby, vous devez exécuter la commande bundler en tant qu'utilisateur non root avec les privilèges sudo. Par conséquent, créez un utilisateur Redmine affecté au répertoire d’installation de Redmine, /opt/redmine en tant que répertoire home.

```bash
useradd -r -m -d /opt/redmine -s /bin/bash redmine
```

Ajoutez l'utilisateur Apache au groupe Redmine.

```bash
usermod -aG redmine www-data
```

### Installer MariaDB sur Debian 10 Buster

```bash
apt install mariadb-server mariadb-client
```

#### Créer une base de données Redmine et un utilisateur de base de données

Une fois MariaDB installé, connectez-vous en tant qu'utilisateur root et créez la base de données Redmine et l'utilisateur de la base de données. Remplacez les noms de la base de données et de l'utilisateur de la base de données en conséquence.

```bash
mysql -u root -p

CREATE DATABASE redminedb CHARACTER SET utf8mb4 COLLATE = utf8mb4_unicode_ci;
grant all on redminedb.* to redmineuser@localhost identified by 'P@ssW0rD';

#### Reload privilege tables and exit the database.

flush privileges;
quit
```

### Téléchargez et installez Redmine

Redmine v4.0.5 est la dernière version à ce jour. Naviguez sur la page des versions de Redmine et récupérez l’archive Redmine. Vous pouvez simplement le télécharger en exécutant la commande ci-dessous.

```bash
apt install wget
wget http://www.redmine.org/releases/redmine-4.0.5.tar.gz -P /tmp/
```

Extrayez l'archive Redmine dans le répertoire Redmine.

```bash
sudo -u redmine tar xzf /tmp/redmine-4.0.5.tar.gz -C /opt/redmine/ --strip-components=1
```

### Configuration de Redmine sur Debian 10

Une fois que vous avez installé Redmine dans le répertoire /opt/redmine, vous pouvez maintenant procéder à sa configuration.

Créez le fichier de configuration Redmine en renommant les exemples de fichiers de configuration comme indiqué ci-dessous:

```bash
su - redmine

cp /opt/redmine/config/configuration.yml{.example,}

cp /opt/redmine/public/dispatch.fcgi{.example,}

cp /opt/redmine/config/database.yml{.example,}
```

### Configurer les paramètres de la base de données Redmine

Ouvrez le paramètre de configuration de la base de données Redmine créé et définissez les détails de connexion à la base de données Redmine pour MySQL.


```bash
vim /opt/redmine/config/database.yml

...
production:
  adapter: mysql2
  database: redminedb
  host: localhost
  username: redmineuser
  password: "P@ssW0rD"
  encoding: utf8mb4
...
```

### Installer les dépendances de Redmine Ruby

Quittez l'utilisateur Redmine en appuyant sur Ctlr + D et, en tant qu'utilisateur root, accédez au répertoire d'installation de Redmine et installez les dépendances Ruby.

```bash
cd /opt/redmine
```

Installez Bundler pour gérer les dépendances gem.

```bash
gem install bundler
```

Ensuite, installez les dépendances gems requises en tant qu’utilisateur redmine non privilégié.

```bash
su - redmine

bundle install --without development test --path vendor/bundle
```

### Générer un jeton de session secret

Pour empêcher la modération des cookies stockant les données de session, vous devez générer une clé secrète aléatoire que Rails utilise pour les coder.

```bash
bundle exec rake generate_secret_token
```

### Create Database Schema Objects

Create Rails database structure by running the command below;

```bash
RAILS_ENV=production bundle exec rake db:migrate
```

Une fois la migration de la base de données terminée, insérez les données de configuration par défaut dans la base de données en exécutant:

```bash
RAILS_ENV=production REDMINE_LANG=en bundle exec rake redmine:load_default_data
```

### Configure FileSystem Permissions

Assurez-vous que les répertoires suivants sont disponibles dans le répertoire Redmine, /opt/redmine.

```bash
    tmp and tmp/pdf
    public and public/plugin_assets
    log
    files
```

S'ils n'existent pas, créez-les simplement et assurez-vous qu'ils appartiennent à l'utilisateur utilisé pour exécuter Redmine.

```bash
for i in tmp tmp/pdf public/plugin_assets; do [ -d $i ] || mkdir -p $i; done

chown -R redmine:redmine files log tmp public/plugin_assets

chmod -R 755 /opt/redmine
```

### Configure Apache for Redmine


Créez le fichier de configuration Redmine Apache VirtualHost.

```bash
nano /etc/apache2/sites-available/redmine.conf

Listen 3000
<VirtualHost *:3000>
	ServerName redmine.example.com
	RailsEnv production
	DocumentRoot /opt/redmine/public

	<Directory "/opt/redmine/public">
	        Allow from all
	        Require all granted
	</Directory>

	ErrorLog ${APACHE_LOG_DIR}/redmine_error.log
        CustomLog ${APACHE_LOG_DIR}/redmine_access.log combined
</VirtualHost>
```

Vérifiez la configuration d'Apache pour les erreurs.

```bash
apachectl configtest
 Syntax OK
```

Assurez-vous que le module Passager est chargé:

```bash
apache2ctl -M | grep -i passenger

passenger_module (shared)
```

S'il n'est pas activé, exécutez la commande ci-dessous pour l'activer.

```bash
a2enmod passenger
```

Enable Redmine site.

```bash
a2ensite redmine
```

```bash
Enabling site redmine.
To activate the new configuration, you need to run:
  systemctl reload apache2

Reload Apache

systemctl reload apache2
```
