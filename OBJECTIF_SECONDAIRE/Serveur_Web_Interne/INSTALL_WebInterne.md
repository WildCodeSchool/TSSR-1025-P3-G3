# Installation et configuration du serveur web interne

## Prérequis techniques

| Élément      | Valeur             |
| ------------ | ------------------ |
| Machine      | SRVLX02            |
| OS           | Debian 12 Bookworm |
| RAM          | 2 Go               |
| CPU          | 2                  |
| Stockage     | 20 Go              |
| Réseau       | LAN                |
| IP           | 192.168.10.40/24   |
| Passerelle   | 192.168.10.254     |
| DNS          | 192.168.10.5       |
| Hostname     | intranet.tssr.lan  |
| Compte       | root               |
| Mot de passe | Azerty1*           |

---
## Configuration

### Paramètres à configurer

| Paramètre            | Valeur                             |
| -------------------- | ---------------------------------- |
| Serveur web          | Apache 2                           |
| Base de données      | MariaDB                            |
| Langage              | PHP 8.2                            |
| CMS                  | WordPress                          |
| URL Intranet         | http://intranet.tssr.lan           |
| Compte admin WP      | admin                              |
| Mot de passe WP      | Azerty1*                           |
| Titre du site        | Ekoloclast — Intranet              |

---

## Étapes d'installation et configuration

### Création de la VM

1. Ouvrir VirtualBox
2. Cliquer sur **Nouvelle**
3. Configurer :
   - **Nom** : SRVLX02
   - **Type** : Linux
   - **Version** : Debian (64-bit)
   - **RAM** : 2048 Mo
   - **Disque** : 20 Go
4. Aller dans **Configuration** → **Réseau**
5. Adapter 1 : **Réseau interne** → **intnet-lan**

---

### Installation de Debian 12

1. Télécharger l'ISO : https://cdimage.debian.org/cdimage/archive/12.9.0/amd64/iso-cd/debian-12.9.0-amd64-netinst.iso

2. Démarrer la VM et installer avec les paramètres suivants :

| Paramètre              | Valeur                                       |
| ---------------------- | -------------------------------------------- |
| Langue                 | Français                                     |
| Pays                   | France                                       |
| Clavier                | Français                                     |
| Nom de machine         | intranet                                     |
| Domaine                | tssr.lan                                     |
| Mot de passe root      | Azerty1*                                     |
| Utilisateur            | wilder                                       |
| Mot de passe           | Azerty1*                                     |
| Partitionnement        | Disque entier avec LVM                       |
| Sélection logiciels    | Serveur SSH + Utilitaires système uniquement |

**ATTENTION** : NE PAS installer d'interface graphique.

---

### Configuration réseau

1. Éditer le fichier de configuration :

    nano /etc/network/interfaces

2. Contenu :

    auto lo
    iface lo inet loopback

    auto enp0s3
    iface enp0s3 inet static
        address 192.168.10.40
        netmask 255.255.255.0
        gateway 192.168.10.254

3. Configurer le DNS :

    nano /etc/resolv.conf

4. Contenu :

    search tssr.lan
    nameserver 192.168.10.5

5. Protéger le fichier DNS :

    chattr +i /etc/resolv.conf

6. Configurer le hostname :

    hostnamectl set-hostname intranet.tssr.lan

7. Éditer le fichier hosts :

    nano /etc/hosts

8. Contenu :

    127.0.0.1       localhost
    192.168.10.40   intranet.tssr.lan intranet

9. Redémarrer le réseau :

    systemctl restart networking

10. Vérifier :

    ip a
    hostname -f
    ping -c 3 192.168.10.5
    ping -c 3 192.168.10.254

---

### Installation de la stack LAMP

#### Apache

1. Mettre à jour le système :

    apt update && apt upgrade -y

2. Installer Apache :

    apt install -y apache2

3. Vérifier le service :

    systemctl status apache2

4. Tester depuis un client Windows en ouvrant un navigateur sur : http://192.168.10.40

La page par défaut d'Apache doit s'afficher.

---

#### MariaDB

1. Installer MariaDB :

    apt install -y mariadb-server mariadb-client

2. Sécuriser l'installation :

    mysql_secure_installation

3. Répondre aux questions :

| Question                             | Réponse  |
| ------------------------------------ | -------- |
| Enter current password for root      | Entrée   |
| Switch to unix_socket authentication | n        |
| Change the root password             | y        |
| New password                         | Azerty1* |
| Remove anonymous users               | y        |
| Disallow root login remotely         | y        |
| Remove test database                 | y        |
| Reload privilege tables              | y        |

4. Créer la base de données WordPress :

    mysql -u root -p

    CREATE DATABASE wordpress DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
    CREATE USER 'wpuser'@'localhost' IDENTIFIED BY 'Azerty1*';
    GRANT ALL PRIVILEGES ON wordpress.* TO 'wpuser'@'localhost';
    FLUSH PRIVILEGES;
    EXIT;

---

#### PHP

1. Installer PHP et les extensions nécessaires :

    apt install -y php php-mysql php-curl php-gd php-mbstring php-xml php-xmlrpc php-soap php-intl php-zip php-imagick

2. Vérifier la version :

    php -v

Le résultat doit afficher PHP 8.2.x.

3. Redémarrer Apache :

    systemctl restart apache2

---

### Installation de WordPress

1. Télécharger WordPress :

    cd /tmp
    wget https://fr.wordpress.org/latest-fr_FR.tar.gz
    tar -xzf latest-fr_FR.tar.gz

2. Copier dans le répertoire web :

    cp -r /tmp/wordpress/* /var/www/html/
    rm /var/www/html/index.html

3. Configurer les permissions :

    chown -R www-data:www-data /var/www/html/
    chmod -R 755 /var/www/html/

4. Configurer WordPress :

    cd /var/www/html
    cp wp-config-sample.php wp-config.php
    nano wp-config.php

5. Modifier les lignes suivantes :

    define( 'DB_NAME', 'wordpress' );
    define( 'DB_USER', 'wpuser' );
    define( 'DB_PASSWORD', 'Azerty1*' );
    define( 'DB_HOST', 'localhost' );

6. Générer les clés de sécurité en allant sur : https://api.wordpress.org/secret-key/1.1/salt/

7. Copier les clés générées et remplacer les lignes correspondantes dans wp-config.php

8. Ajouter en fin de fichier avant la ligne "That's all, stop editing!" :

    define( 'FS_METHOD', 'direct' );

9. Sauvegarder et fermer

---

### Configuration du VirtualHost Apache

1. Créer le fichier de configuration :

    nano /etc/apache2/sites-available/intranet.conf

2. Contenu :

    <VirtualHost *:80>
        ServerName intranet.tssr.lan
        DocumentRoot /var/www/html

        <Directory /var/www/html>
            AllowOverride All
            Require all granted
        </Directory>

        ErrorLog ${APACHE_LOG_DIR}/intranet_error.log
        CustomLog ${APACHE_LOG_DIR}/intranet_access.log combined
        
    </VirtualHost>

3. Activer le site et les modules nécessaires :

    a2ensite intranet.conf
    a2dissite 000-default.conf
    a2enmod rewrite
    systemctl restart apache2

---

### Finalisation WordPress via navigateur

1. Depuis un client Windows, ouvrir un navigateur
2. Accéder à : http://192.168.10.40
3. Remplir le formulaire d'installation :

| Champ              | Valeur               |
| ------------------ | -------------------- |
| Titre du site      | Ekoloclast  Intranet |
| Identifiant        | admin                |
| Mot de passe       | Azerty1*             |
| Adresse email      | postmaster@tssr.lan  |
| Visibilité moteurs | Cocher (bloquer)     |

4. Cliquer sur **Installer WordPress**
5. Se connecter à l'interface d'administration : http://192.168.10.40/wp-admin

---

### Enregistrement DNS sur SRVWIN01

**Ne pas oublier cette étape pour que l'accès par nom fonctionne.**

1. Sur **SRVWIN01**, ouvrir **Server Manager** → **Tools** → **DNS**
2. Développer **Forward Lookup Zones** → **tssr.lan**
3. Clic droit → **New Host (A or AAAA)**
4. Créer l'enregistrement :

| Nom       | IP             |
| --------- | -------------- |
| intranet  | 192.168.10.40  |

5. Cliquer sur **Add Host**

---

### Vérification

1. Depuis SRVLX02 :

    systemctl status apache2
    systemctl status mariadb
    curl -I http://localhost

2. Depuis un client Windows :
   - Ouvrir un navigateur
   - Accéder à http://intranet.tssr.lan
   - Vérifier que WordPress s'affiche

3. Vérification DNS :

    nslookup intranet.tssr.lan

Le résultat doit résoudre vers 192.168.10.40.

---

## FAQ

### La page Apache par défaut s'affiche au lieu de WordPress

Vérifier que le fichier index.html par défaut a bien été supprimé :

    ls /var/www/html/index.html

S'il existe encore :

    rm /var/www/html/index.html

### Erreur "Error establishing a database connection"

Vérifier les identifiants dans wp-config.php :

    cat /var/www/html/wp-config.php | grep DB_

Vérifier que MariaDB est démarré :

    systemctl status mariadb

Tester la connexion :

    mysql -u wpuser -p wordpress

### Les permaliens WordPress ne fonctionnent pas (erreur 404)

Vérifier que le module rewrite est activé :

    a2enmod rewrite
    systemctl restart apache2

Vérifier que AllowOverride est sur All dans le VirtualHost.

### Le fichier resolv.conf se réinitialise au redémarrage

Protéger le fichier avec chattr :

    chattr +i /etc/resolv.conf

Pour modifier ultérieurement :

    chattr -i /etc/resolv.conf
    nano /etc/resolv.conf
    chattr +i /etc/resolv.conf

### Impossible d'accéder au site depuis les clients Windows

Vérifier que le client est sur le bon réseau (192.168.10.x) :

    ipconfig

Vérifier la résolution DNS :

    nslookup intranet.tssr.lan

Si la résolution échoue, vérifier les enregistrements DNS sur SRVWIN01.

### WordPress demande les identifiants FTP pour installer des plugins

Ajouter dans wp-config.php :

    define( 'FS_METHOD', 'direct' );

Vérifier les permissions :

    chown -R www-data:www-data /var/www/html/