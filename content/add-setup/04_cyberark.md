# Installation de conjur cyberark sur openshift

# Prerequisites 

Installez le conjur cli.
Avoir metal Lb ou un autres service de loadbalancing disponible. 
NOTE: Il est surement possible de la faire via une route ,mais cela n'est pas traité dans ce guide.

## Installation de cyberark

Pour commencer nous allons installer conjur sur Openshift dans un namespace dédié.

```shell
oc new-project conjur
```

Pour cela un Helm chart est mis à disposition. Cloner le repo ci-dessous.

```shell
git clone TOCOMPLETE
```
Puis installez conjur 

```shell
DATA_KEY=$(docker run --rm cyberark/conjur data-key generate)
helm install conjur ./conjur-oss --set dataKey="${DATA_KEY}" --set authenticators="authn\,authn-k8s/conjur" --version 2.0.0 --set openshift.enabled=true --namespace conjur
#Exemple 
#helm install conjur ./conjur-oss --set dataKey="8QYoky5p9szd9iXxSRmoOqGClOebfBPNCEpBl3Bn7jo=" --set authenticators="authn\,authn-k8s/conjur" --version 2.0.0 --set openshift.enabled=true --namespace conjur
```

On configure ici 2 authenticators : ```authn``` pour une authentification en login/password standard et ```authn-k8s``` pour l'authentification des pods.

On créer maintenant un compte par défaut :

```shell
oc get po # On récupere ici le nom du pods
oc exec -it conjur-conjur-oss-6466669767-zbk6n -c conjur-oss -- conjurctl account create default
```

**NOTE: Conserver l'ouput de la commande précédente. Le mot de passe sera utilisé dans la suite de ce guide.**

Dans notre helm chart le certificat est configuré pour le nom de domaine conjur.myorg.com. On va donc éditer le local base domain.

```shell
oc get svc conjur-conjur-oss-ingress -ojsonpath='{.status.loadBalancer.ingress[0].ip}'
#10.253.234.69
```
```shell
sudo vi /etc/hosts
# add conjur.myorg.com 10.253.234.69
```

On va ensuite utiliser la commande ```conjur init``` pour créer un fichier de configuration (.conjurrc) qui contient les détails de connexion à Conjur. Ce fichier se trouvera alors  sous le répertoire racine de l'utilisateur.

```shell
conjur init --url=https://conjur.myorg.com --account=default -s

#The Conjur server's certificate SHA-1 fingerprint is:
#XX:XX:XX

#To verify this certificate, we recommend running the following command on the Conjur server:
#openssl x509 -fingerprint -noout -in ~conjur/etc/ssl/conjur.pem

#Trust this certificate? yes/no (Default: no): yes
#File /home/youruser/conjur-server.pem exists. Overwrite? yes/no (Default: yes): yes
#Certificate written to /home/feven/conjur-server.pem

#File /home/youruser/.conjurrc exists. Overwrite? yes/no (Default: yes): yes
#Configuration written to /home/youruser/.conjurrc

#Successfully initialized the Conjur CLI
#To start using the Conjur CLI, log in to the Conjur server by running `conjur login`
```

Vous pouvez maintenant vous connectez au serveur conjur. Le mot de passe est l'API key for admin généré lors des étapes précédentes.

```shell
conjur login
#Enter your username: admin
#Enter your password or API key (this will not be echoed):
#WARNING:root:No supported keystore found! Saving credentials in plaintext in '/home/feven/.netrc'. #Make sure to logoff after you have finished using the CLI
#Successfully logged in to Conjur
```

## Gestion des token pour Openshift










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






