# Installation de wordpress sur Scaleway

## Configuration de l'instance

Pour commencer, cliquez sur Compute > Instances > Create Instance.

Completez le formulaire avec les valeurs ci-dessous.

|Settings            | Value   |
| ------------------ | ------- |
| Availability Zone  | Paris 1 |
| Select an Instance | Dev1-S  |
| Choose an image    | Fedora  |

Ajoutez ensuite votre clé ssh et cliquez sur Créer une instance.

## Installation d'Apache et de PHP

Connectez-vous à votre instance en utilisant ssh et suivez les instructions ci-dessous.

Tout d’abord, installez le serveur Apache et php avec la commande suivante.

```shell
sudo dnf update -y
sudo dnf install httpd -y
sudo systemctl start httpd
sudo systemctl enable httpd
sudo dnf install php php-fpm php-mysqlnd php-opcache php-gd php-xml php-mbstring php-curl php-pecl-imagick php-pecl-zip libzip -y
```

Ensuite, éditez le fichier de configuration PHP et modifiez les paramètres par défaut adaptés à vos besoins :

```shell
sudo vi /etc/php.ini
```

```shell
max_execution_time = 300 
max_input_time = 300 
memory_limit = 512M 
post_max_size = 256M 
upload_max_filesize = 256M
```

Enregistrez et fermez le fichier, puis démarrez et activez le service PHP-FPM avec la commande suivante.


```shell
sudo systemctl start php-fpm
sudo systemctl enable php-fpm
```

## Installer et configurer la base de données MariaDB

Tout d’abord, installez le serveur MariaDB avec la commande suivante.

```shell
sudo dnf install mariadb-server -y
sudo systemctl start mariadb
sudo systemctl enable mariadb
```

Ensuite, connectez-vous au shell MariaDB et créez une base de données et un utilisateur pour WordPress.

NOTE: Update wpuser, localhost et password avec vos propres valeurs.

```shell
mysql
MariaDB [(none)]> CREATE DATABASE wpdb;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON wpdb.* TO 'wpuser'@'localhost' IDENTIFIED BY 'password';
MariaDB [(none)]> FLUSH PRIVILEGES;
MariaDB [(none)]> EXIT;
```

## Téléchargez WordPress

Téléchargez la dernière version de WordPress avec la commande suivante.

```shell
cd /var/www/html/
wget https://www.wordpress.org/latest.tar.gz
tar -xvf latest.tar.gz 
cd wordpress
cp wp-config-sample.php wp-config.php
```

Ensuite, modifiez le fichier de configuration WordPress :

```shell
sudo vi  wp-config.php
```

```shell
/** The name of the database for WordPress */
define( 'DB_NAME', 'wpdb' );

/** Database username */
define ('DB_USER', 'wpuser');

/** Database password */
define( 'DB_PASSWORD', 'password' );

/** Database hostname */
define( 'DB_HOST', 'localhost' );
```

Enregistrez et fermez le fichier, puis modifiez l'autorisation et la propriété du répertoire WordPress.

```shell
chown -R apache:apache /var/www/html/wordpress
chmod -R 755 /var/www/html/wordpress
```

Ensuite, vous devrez créer un hôte virtuel Apache pour définir votre répertoire et votre domaine WordPress.


```shell
vi /etc/httpd/conf.d/wp.conf
```

```shell
<VirtualHost *:80>
    ServerAdmin admin@example.com
    ServerName wp.example.com
    DocumentRoot /var/www/html/wordpress
    <Directory /var/www/html/wordpress>
        Allowoverride all
    </Directory>
</VirtualHost>
```

Enregistrez et fermez le fichier, puis redémarrez le service Apache pour appliquer les modifications.

```shell
systemctl restart httpd 
```






