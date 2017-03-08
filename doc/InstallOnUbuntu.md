# Install Packagist on Ubuntu 16.04
This installation guide assumes the following:
* A basic Ubuntu 16.04 install.
* The installation path for Packagist is `/srv/packagist/`.
* The composer home is `/srv/composer/`.
* Packagist will run on a separate `packagist` user.


## 1. Install APT packages
The commands below install the following packages:
* Apache 2.4.
* PHP 7.1 and required modules.
* MySQL server.
* Version control binaries Git and Subversion.
* Postfix: a Mail Tranfer Agent.

```
sudo add-apt-repository ppa:ondrej/php
sudo add-apt-repository ppa:chris-lea/redis-server
sudo apt-get update
sudo apt-get install -y postfix git subversion mysql-server default-jdk redis-server solr-tomcat libapache2-mpm-itk libapache2-mod-php7.1 php7.1-cli php7.1-curl php7.1-intl php7.1-json php7.1-mysql php7.1-xml php7.1-zip
```


## 2. Create the Packagist user
```
sudo useradd -g www-data -M -d /srv/packagist packagist
sudo mkdir /srv/packagist
sudo -i -u packagist ssh-keygen
```

Add the generated SSH public key to the user in [GitHub](https://github.com/settings/keys).

:warning: It's advisable create a separate user for managing the packages. This way you can limit the access on the repositories.


## 3. Configure Apache 2.4
```
sudo a2enmod rewrite
sudo service apache2 restart
```

Create an Apache configuration:
```
<VirtualHost *:80>
    ServerName composer.packages

    AssignUserId packagist www-data

    DocumentRoot /srv/packagist/production/web/
    DirectoryIndex app.php

    <Directory /srv/packagist/production/web>
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/error.composer.packages.log
    CustomLog ${APACHE_LOG_DIR}/access.composer.packages.log combined
</VirtualHost>
```


## 4. Configure MySQL server 5.7
Create a database and a database user.

Change the SQL mode by adding the following configuration:
```
sudo echo "[mysqld]" > /etc/mysql/mysql.conf.d/packagist.cnf
sudo echo "sql-mode=\"STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION\"" >> /etc/mysql/mysql.conf.d/packagist.cnf
sudo service mysql restart
```


## 5. Install Composer
```
curl -sS https://getcomposer.org/installer | php
sudo mv composer.phar /usr/local/bin/composer
```


## 6. Install Packagist

### Clone from the Connect Holland GitHub fork
```
git clone -b customized --single-branch git@github.com:ConnectHolland/packagist.git /srv/packagist/customized
```

### Configure parameters.yml
```
cp /srv/packagist/customized/app/config/parameters.yml.dist /srv/packagist/customized/app/config/parameters.yml
```

Configure the `parameters.yml` with the relevant values for your setup.

### Install Composer dependencies
```
composer install --no-dev -o
```

### Fix permissions
```
HTTPDUSER=`ps axo user,comm | grep -E '[a]pache|[h]ttpd|[_]www|[w]ww-data|[n]ginx' | grep -v root | head -1 | cut -d\  -f1`

sudo setfacl -R -m u:"$HTTPDUSER":rwX -m u:`whoami`:rwX app/cache app/logs
sudo setfacl -dR -m u:"$HTTPDUSER":rwX -m u:`whoami`:rwX app/cache app/logs
```

For more information, see: [Setting up or Fixing File Permissions](http://symfony.com/doc/2.8/setup/file_permissions.html)

### Warmup cache
```
/srv/packagist/customized/app/console cache:warmup --env=prod
```

### Install assets
```
/srv/packagist/customized/app/console assets:install --env=prod
```

### Setup the database
```
/srv/packagist/customized/app/console doctrine:schema:create
```

### Create the symlink for production
```
ln -s /srv/packagist/customized/ /srv/packagist/production
```


## 7. Configure Solr 3.6

### Copy the schema provided with Packagist
```
sudo cp /etc/solr/conf/schema.xml /etc/solr/conf/schema.xml.bak
sudo cp /srv/packagist/customized/doc/schema.xml /etc/solr/conf/schema.xml
```

### Configure the core in solr.xml
Change the core 'name' attribute and 'defaultCoreName' attribute to 'packagist' in `/etc/solr/solr.xml`.
```
sudo service tomcat7 restart
```


## 8. Configure commands in crontab
Running the update and dump commands for the first time requires them to run in forced mode:
```
/srv/packagist/production/app/console packagist:update --env=prod --force
/srv/packagist/production/app/console packagist:dump --env=prod --force
```

Afterwards you can install the commands in the crontab:
```
sudo echo "" >> /etc/crontab
sudo echo "*   * *   *   *   packagist   /srv/packagist/production/app/console packagist:update --no-debug --env=prod" >> /etc/crontab
sudo echo "*   * *   *   *   packagist   /srv/packagist/production/app/console packagist:dump --no-debug --env=prod" >> /etc/crontab
sudo echo "*   * *   *   *   packagist   /srv/packagist/production/app/console packagist:index --no-debug --env=prod" >> /etc/crontab
sudo echo "0   2 *   *   *   packagist   /srv/packagist/production/app/console packagist:stats:compile --no-debug --env=prod" >> /etc/crontab
```
