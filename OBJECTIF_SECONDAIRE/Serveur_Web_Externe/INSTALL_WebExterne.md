# Installation et configuration du serveur web externe (DMZ)

## Prérequis techniques

| Élément      | Valeur              |
| ------------ | ------------------- |
| Machine      | SRVLX03             |
| OS           | Debian 12 Bookworm  |
| RAM          | 1 Go                |
| CPU          | 1                   |
| Stockage     | 10 Go               |
| Réseau       | DMZ                 |
| IP           | 192.168.30.5/24     |
| Passerelle   | 192.168.30.254      |
| DNS          | 192.168.10.5        |
| Hostname     | webexterne.tssr.lan |
| Compte       | root                |
| Mot de passe | Azerty1*            |

---

## Configuration

### Paramètres à configurer

| Paramètre         | Valeur                                  |
| ----------------- | --------------------------------------- |
| Serveur web       | Apache 2                                |
| Type de site      | HTML statique (vitrine)                 |
| URL interne (LAN) | http://192.168.30.5                     |
| URL externe (WAN) | http://IP_WAN_FW01 http://192.168.1.38/ |

**Note** : L'IP WAN de FW01 est attribuée en DHCP. Vérifier l'IP actuelle dans **Status** → **Interfaces** → **WAN** sur pfSense.

---

## Étapes d'installation et configuration

### Création de la VM

1. Ouvrir VirtualBox
2. Cliquer sur **Nouvelle**
3. Configurer :
   - **Nom** : SRVLX03
   - **Type** : Linux
   - **Version** : Debian (64-bit)
   - **RAM** : 1024 Mo
   - **Disque** : 10 Go
4. Aller dans **Configuration** → **Réseau**
5. Adapter 1 : **Réseau interne** → **DMZ**

**IMPORTANT** : Le nom du réseau interne doit être exactement le même que celui de l'Adapter 3 (DMZ) de FW01. Vérifier dans VirtualBox → FW01 → Configuration → Réseau → Adapter 3 → Name.

---

### Installation de Debian 12

1. Télécharger l'ISO : https://cdimage.debian.org/cdimage/archive/12.9.0/amd64/iso-cd/debian-12.9.0-amd64-netinst.iso

2. Démarrer la VM et installer avec les paramètres suivants :

| Paramètre              | Valeur                                       |
| ---------------------- | -------------------------------------------- |
| Langue                 | Français                                     |
| Pays                   | France                                       |
| Clavier                | Français                                     |
| Nom de machine         | webexterne                                   |
| Domaine                | tssr.lan                                     |
| Mot de passe root      | Azerty1*                                     |
| Utilisateur            | wilder                                       |
| Mot de passe           | Azerty1*                                     |
| Partitionnement        | Disque entier avec LVM                       |
| Sélection logiciels    | Serveur SSH + Utilitaires système uniquement |

**ATTENTION** : NE PAS installer d'interface graphique.

**ATTENTION** : L'installation nécessite un accès Internet pour télécharger les paquets. Si la VM est en DMZ sans accès Internet, basculer temporairement l'adaptateur réseau sur **NAT** le temps de l'installation, puis remettre sur **DMZ** après.

---

### Configuration réseau

1. Éditer le fichier de configuration :

    nano /etc/network/interfaces

2. Contenu :

    auto lo
    iface lo inet loopback

    auto enp0s3
    iface enp0s3 inet static
        address 192.168.30.5
        netmask 255.255.255.0
        gateway 192.168.30.254

3. Configurer le DNS :

    nano /etc/resolv.conf

4. Contenu :

    search tssr.lan
    nameserver 192.168.10.5

5. Protéger le fichier DNS :

    chattr +i /etc/resolv.conf

6. Configurer le hostname :

    hostnamectl set-hostname webexterne.tssr.lan

7. Éditer le fichier hosts :

    nano /etc/hosts

8. Contenu :

    127.0.0.1       localhost
    192.168.30.5    webexterne.tssr.lan webexterne

9. Redémarrer le réseau :

    systemctl restart networking

10. Vérifier :

    ip a
    hostname -f

**Note** : Le ping vers la passerelle (192.168.30.254) ne fonctionnera pas car les règles DMZ n'autorisent pas ICMP. C'est un comportement normal. Pour vérifier la connectivité, utiliser la commande suivante après configuration des règles pfSense :

    apt update

---

### Installation d'Apache

1. Mettre à jour le système :

    apt update && apt upgrade -y

Si apt ne fonctionne pas (pas d'accès Internet depuis la DMZ), configurer d'abord les règles pare-feu pfSense (voir section suivante), puis revenir à cette étape.

2. Installer Apache :

    apt install -y apache2

3. Vérifier le service :

    systemctl status apache2

4. Activer Apache au démarrage :

    systemctl enable apache2

---

### Déploiement du site vitrine

1. Supprimer la page par défaut :

    rm /var/www/html/index.html

2. Créer le fichier du site vitrine :

    nano /var/www/html/index.html

3. Coller le contenu du fichier vitrine.html

4. Configurer les permissions :

    chown -R www-data:www-data /var/www/html/
    chmod -R 755 /var/www/html/

---

## Configuration des règles pare-feu pfSense

### Étape 1 : Règles DMZ sortantes 

Autoriser le serveur DMZ à accéder à Internet pour les mises à jour système.

1. Se connecter à pfSense : https://192.168.10.254
2. Aller dans **Firewall** → **Rules** → **DMZ**
3. Cliquer sur **Add** (flèche vers le bas)
4. Créer les règles suivantes :

| Action | Source       | Destination | Port | Protocol | Description                      |
| ------ | ------------ | ----------- | ---- | -------- | -------------------------------- |
| Pass   | 192.168.30.5 | Any         | 80   | TCP      | Serveur web DMZ → Internet HTTP  |
| Pass   | 192.168.30.5 | Any         | 443  | TCP      | Serveur web DMZ → Internet HTTPS |
| Pass   | 192.168.30.5 | Any         | 53   | TCP/UDP  | Serveur web DMZ → DNS            |

Pour chaque règle :
   - **Action** : Pass
   - **Interface** : DMZ
   - **Protocol** : TCP (ou TCP/UDP pour DNS)
   - **Source** : Address or Alias — 192.168.30.5
   - **Destination** : Any
   - **Destination port range** : selon le tableau
   - **Description** : selon le tableau
5. Cliquer sur **Save** après chaque règle
6. Cliquer sur **Apply Changes**

---

### Étape 2 : NAT Port Forwarding (WAN → DMZ)

Rediriger le trafic HTTP et HTTPS du WAN vers le serveur web DMZ.

1. Aller dans **Firewall** → **NAT** → **Port Forward**
2. Cliquer sur **Add** (flèche vers le bas)

#### Règle HTTP :

| Paramètre              | Valeur                        |
| ---------------------- | ----------------------------- |
| Interface              | WAN                           |
| Protocol               | TCP                           |
| Destination            | WAN Address                   |
| Destination port range | HTTP (80)                     |
| Redirect target IP     | 192.168.30.5                  |
| Redirect target port   | HTTP (80)                     |
| Description            | NAT HTTP vers serveur web DMZ |

3. Cliquer sur **Save**

#### Règle HTTPS :

| Paramètre              | Valeur                         |
| ---------------------- | ------------------------------ |
| Interface              | WAN                            |
| Protocol               | TCP                            |
| Destination            | WAN Address                    |
| Destination port range | HTTPS (443)                    |
| Redirect target IP     | 192.168.30.5                   |
| Redirect target port   | HTTPS (443)                    |
| Description            | NAT HTTPS vers serveur web DMZ |

4. Cliquer sur **Save**
5. Cliquer sur **Apply Changes**

**Note** : pfSense crée automatiquement les règles firewall WAN associées lors de la création du NAT Port Forward.

---

### Étape 3 : Débloquer les réseaux privés sur l'interface WAN

En environnement de lab, l'interface WAN utilise une adresse IP privée (192.168.1.x). Par défaut pfSense bloque le trafic provenant de réseaux privés sur le WAN. Il faut désactiver ce blocage pour que l'accès depuis le PC hôte fonctionne.

1. Aller dans **Interfaces** → **WAN**
2. Descendre tout en bas jusqu'à la section **Reserved Networks**
3. **Décocher** la case "Block private networks and loopback addresses"
4. Laisser "Block bogon networks" coché
5. Cliquer sur **Save**
6. Cliquer sur **Apply Changes**

**Note** : Cette modification est nécessaire uniquement en environnement de lab où le WAN utilise des adresses privées. En production avec une IP publique, cette case resterait cochée.

---

### Étape 4 : Activer le NAT Reflection

Pour que le site soit accessible depuis le WAN via l'IP WAN de pfSense, il faut activer le NAT Reflection.

1. Aller dans **System** → **Advanced** → **Firewall & NAT**
2. Chercher la section **Network Address Translation**
3. **NAT Reflection mode for port forwards** : sélectionner **Pure NAT**
4. Cocher **Enable NAT Reflection for 1:1 NAT**
5. Cocher **Enable automatic outbound NAT for Reflection**
6. Cliquer sur **Save**

---

### Étape 5 : Accès LAN vers DMZ

Le LAN a déjà une règle pass-all vers Any, donc les clients du LAN peuvent accéder au serveur web DMZ sans règle supplémentaire.

Vérifier depuis un client Windows : http://192.168.30.5

---

### Enregistrement DNS sur SRVWIN01 (optionnel)

Permet d'accéder au site depuis le LAN via http://webexterne.tssr.lan au lieu de l'IP.

1. Sur **SRVWIN01**, ouvrir **Server Manager** → **Tools** → **DNS**
2. Développer **Forward Lookup Zones** → **tssr.lan**
3. Clic droit → **New Host (A or AAAA)**
4. Créer l'enregistrement :

| Nom         | IP             |
| ----------- | -------------- |
| webexterne  | 192.168.30.5   |

5. Cliquer sur **Add Host**

---

## Vérification

### Depuis SRVLX03

    systemctl status apache2
    curl -I http://localhost

Le résultat doit afficher HTTP/1.1 200 OK.

### Depuis un client Windows (LAN)

1. Ouvrir un navigateur
2. Accéder à http://192.168.30.5
3. Vérifier que le site vitrine Ekoloclast s'affiche

### Depuis l'extérieur (WAN)

1. Vérifier l'IP WAN actuelle de FW01 :
   - Sur pfSense → **Status** → **Interfaces** → **WAN** → **IPv4 Address**
   - L'IP est attribuée en DHCP et peut changer à chaque redémarrage
2. Depuis une machine sur le réseau WAN (ou le PC hôte)
3. Ouvrir un navigateur
4. Accéder à http://IP_WAN (exemple : http://192.168.1.38)
5. Vérifier que le site vitrine Ekoloclast s'affiche

Si le site ne s'affiche pas depuis le WAN, vérifier dans l'ordre :
   - L'IP WAN actuelle dans **Status** → **Interfaces**
   - Le NAT Port Forward dans **Firewall** → **NAT** → **Port Forward**
   - Le déblocage des réseaux privés (Étape 3)
   - Le NAT Reflection (Étape 4)


---

## FAQ

### apt ne fonctionne pas (pas d'accès Internet)

Le serveur est en DMZ, il faut d'abord configurer les règles pare-feu sortantes (Étape 1) pour autoriser l'accès HTTP, HTTPS et DNS vers Internet.

Vérifier la connectivité avec :

    apt update

**Note** : Le ping ne fonctionne pas depuis la DMZ car ICMP n'est pas autorisé dans les règles. C'est normal. Utiliser apt update pour tester la connectivité Internet.

### Le site n'est pas accessible depuis le LAN

Vérifier que le serveur est bien démarré :

    systemctl status apache2

Vérifier que la règle LAN pass-all est en place dans pfSense (par défaut elle existe).

### Le site n'est pas accessible depuis le WAN

Vérifier les points suivants dans l'ordre :

1. L'IP WAN actuelle : **Status** → **Interfaces** → **WAN** → **IPv4 Address** (l'IP change car elle est en DHCP)

2. Le NAT Port Forward : **Firewall** → **NAT** → **Port Forward** (deux règles HTTP et HTTPS doivent être présentes)

3. Le déblocage des réseaux privés : **Interfaces** → **WAN** → **Reserved Networks** → "Block private networks" doit être décoché en environnement de lab

4. Le NAT Reflection : **System** → **Advanced** → **Firewall & NAT** → NAT Reflection mode doit être sur Pure NAT

5. Le serveur web écoute bien sur le port 80 :

    ss -tlnp | grep 80

### Le fichier resolv.conf se réinitialise au redémarrage

Protéger le fichier avec chattr :

    chattr +i /etc/resolv.conf

Pour modifier ultérieurement :

    chattr -i /etc/resolv.conf
    nano /etc/resolv.conf
    chattr +i /etc/resolv.conf

### Le DNS ne résout pas depuis la DMZ

Le serveur DNS est sur le LAN (192.168.10.5). Il faut que la DMZ puisse joindre le LAN pour la résolution DNS.

Alternative : utiliser un DNS public dans resolv.conf :

    nameserver 8.8.8.8
    nameserver 8.8.4.4

### Le ping ne fonctionne pas depuis la DMZ

C'est normal. Les règles DMZ n'autorisent que TCP 80, 443 et DNS 53. Le protocole ICMP (ping) n'est pas autorisé. Utiliser apt update pour vérifier la connectivité Internet.

### Le nom du réseau interne ne correspond pas

Les VM SRVLX03 et FW01 doivent avoir exactement le même nom de réseau interne pour la DMZ dans VirtualBox. Vérifier dans les paramètres réseau de chaque VM que le nom est identique (sensible à la casse).