---
title: Installer Redmine avec MariaDB sur Debian 10 Buster
path: /redmine-setup-debian-buster
date: 2019-11-28
summary: Mise en place de Redmine, setup pour un environnement de production en SSL et d'intégration avec Apache2.
tags: ['Redmine', Gestion de projet', 'Debian 10']
---
### Run system update

To begin with, ensure that your system packages are up-to-date.

```bash
apt update
```

### Install Required Build Tools and Dependencies

To install Redmine from the source code, you need install the required build tools and dependencies.

apt install build-essential ruby-dev libxslt1-dev libmariadb-dev libxml2-dev zlib1g-dev imagemagick libmagickwand-dev curl vim sudo

Install Apache HTTP Server on Debian 10 Buster

Redmine is a web application and hence you need to install a web server to access it.

```bash
apt install apache2
```

Next, install the APache modules for the Passenger, lightweight web server for Ruby.

```bash
apt install libapache2-mod-passenger
```

The above command will also install other required dependencies including Ruby.

```bash
ruby -v

ruby 2.5.5p157 (2019-03-15 revision 67260) [x86_64-linux-gnu]
```

Start and enable Apache to run on system boot.

```bash
systemctl enable --now apache2
```

### Create Redmine System User

While installing the Redmine Ruby dependencies, you need to run the bundler command as a non root user with sudo privileges. Hence, create a redmine user assign the Redmine install directory, /opt/redmine as its home directory.

```bash
useradd -r -m -d /opt/redmine -s /usr/bin/bash redmine
```

Add Apache user to Redmine group.

```bash
usermod -aG redmine www-data
```

### Install MariaDB on Debian 10 Buster

#### Create Redmine Database and Database User

Once MariaDB is installed, login as root user and create Redmine database and database user. Replace the names of the database and the database user accordingly.

```bash
mysql -u root -p

create database redminedb;

grant all on redminedb.* to redmineuser@localhost identified by 'P@ssW0rD';

#### Reload privilege tables and exit the database.

flush privileges;
quit
```

### Download and Install Redmine

Redmine v4.0.5 is the latest release as of this writing. Navigate Redmine releases page and grab Redmine tarball. You can simply download it by running the command below.

```bash
wget http://www.redmine.org/releases/redmine-4.0.5.tar.gz -P /tmp/
```

Extract the Redmine tarball to the Redmine directory.

```bash
sudo -u redmine tar xzf /tmp/redmine-4.0.5.tar.gz -C /opt/redmine/ --strip-components=1
```

### Configuring Redmine on Debian 10

Once you have installed Redmine under the /opt/redmine directory, you can now proceed to configure it.

Create Redmine configuration file by renaming the sample configuration files as shown below;

```bash
su - redmine

cp /opt/redmine/config/configuration.yml{.example,}

cp /opt/redmine/public/dispatch.fcgi{.example,}

cp /opt/redmine/config/database.yml{.example,}
```

### Configure Redmine Database Settings

Open the created Redmine database configuration setting and set the Redmine database connection details for MySQL.

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