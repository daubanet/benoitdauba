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
useradd -r -m -d /opt/redmine -s /usr/bin/bash redmine
```

Ajoutez l'utilisateur Apache au groupe Redmine.

```bash
usermod -aG redmine www-data
```

### Installer MariaDB sur Debian 10 Buster

#### Créer une base de données Redmine et un utilisateur de base de données

Une fois MariaDB installé, connectez-vous en tant qu'utilisateur root et créez la base de données Redmine et l'utilisateur de la base de données. Remplacez les noms de la base de données et de l'utilisateur de la base de données en conséquence.

```bash
mysql -u root -p

create database redminedb;

grant all on redminedb.* to redmineuser@localhost identified by 'P@ssW0rD';

#### Reload privilege tables and exit the database.

flush privileges;
quit
```

### Téléchargez et installez Redmine

Redmine v4.0.5 est la dernière version à ce jour. Naviguez sur la page des versions de Redmine et récupérez l’archive Redmine. Vous pouvez simplement le télécharger en exécutant la commande ci-dessous.

```bash
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
  encoding: utf8
...
```

### Install Redmine Ruby Dependencies

Exit the Redmine user by pressing Ctlr+D and as root user, navigate to Redmine install directory and install the Ruby dependencies.

```bash
cd /opt/redmine
```

Install Bundler for managing gem dependencies.

```bash
gem install bundler
```

Next, install the required gems dependencies as non-privileged redmine user.

```bash
su - redmine

bundle install --without development test --path vendor/bundle
```

### Generate Secret Session Token

To prevent tempering of the cookies that stores session data, you need to generate a random secret key that Rails uses to encode them.

```bash
bundle exec rake generate_secret_token

Create Database Schema Objects

Create Rails database structure by running the command below;

RAILS_ENV=production bundle exec rake db:migrate
```

Once the database migration is done, insert default configuration data into the database by executing;

```bash
RAILS_ENV=production REDMINE_LANG=en bundle exec rake redmine:load_default_data
```

### Configure FileSystem Permissions

Ensure that the following directories are available on Redmine directory, /opt/redmine.
```bash
    tmp and tmp/pdf
    public and public/plugin_assets
    log
    files
```

If they do not exist, simply create them and ensure that they are owned by the user used to run Redmine.

```bash
for i in tmp tmp/pdf public/plugin_assets; do [ -d $i ] || mkdir -p $i; done

chown -R redmine:redmine files log tmp public/plugin_assets

chmod -R 755 /opt/redmine
```

### Testing Redmine Installation

The setup of Redmine on CentOS 8 is now done. Redmine listens on TCP port 3000 by default. Hence, before running the tests, open port 3000/tcp on firewall if it is running.

```bash
ufw allow 3000/tcp
```

You can now test Redmine using WEBrick by executing the command below;

```bash
bundle exec rails server webrick -e production
```

Navigate to the browser and enter the address, http://server-IP-or-Hostname:3000. Replace the server-IP-or-Hostname accordingly.

If all is well, you should land on Redmine web user interface.

### Configure Apache for Redmine

Now that you have confirmed that Redmine is working as expected, proceed to configure Apache to server Redmine.

```bash
Create Redmine Apache VirtualHost configuration file.

vim /etc/apache2/sites-available/redmine.conf

Listen 3000
<VirtualHost *:3000>
	ServerName redmine.kifarunix-demo.com
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

Check Apache configuration for errors.

```bash
apachectl configtest
 Syntax OK
```

Ensure that Passenger module is loaded;

```bash
apache2ctl -M | grep -i passenger

passenger_module (shared)
```

If not enabled, run the command below to enable it.

```bash
a2enmod passenger

Enable Redmine site.

a2ensite redmine

Enabling site redmine.
To activate the new configuration, you need to run:
  systemctl reload apache2

Reload Apache

systemctl reload apache2
```

Check to ensure that Redmine is now listening on port 3000.

```bash
lsof -i :3000

apache2 32538     root    6u  IPv6 101516      0t0  TCP *:3000 (LISTEN)
apache2 32647 www-data    6u  IPv6 101516      0t0  TCP *:3000 (LISTEN)
apache2 32648 www-data    6u  IPv6 101516      0t0  TCP *:3000 (LISTEN)
apache2 32649 www-data    6u  IPv6 101516      0t0  TCP *:3000 (LISTEN)
apache2 32650 www-data    6u  IPv6 101516      0t0  TCP *:3000 (LISTEN)
apache2 32651 www-data    6u  IPv6 101516      0t0  TCP *:3000 (LISTEN)
```