# **Ekoloclast**🌿

![Logo Ekoloclast](Ressources/logo_ekoloclast.png)

## Table des matières 

1. [Présentation du projet](#1-présentation-du-projet) 
2. [Introduction](#2-introduction) 
3. [Choix techniques](#3-choix-techniques) 
4. [Difficultés rencontrées](#4-difficultés-rencontrées) 
5. [Solutions trouvées](#5-solutions-trouvées) 
6. [Améliorations possibles](#6-améliorations-possibles)

## 1. Présentation du projet

Ce projet est le troisième projet réalisé au sein de la **Wild Code School**, dans le cadre d'un bootcamp Technicien Superieur Systèmes et Réseaux.

Ce projet a pour objectif de concevoir et construire une infrastructure réseau complète pour l'entreprise Ekoloclast.

Il s'agit d'un projet individuel réalisé sur une durée de 10 semaines.

### Objectifs principaux

L'infrastructure déployée couvre les services suivants :

- **Pare-feu** : segmentation réseau WAN / LAN / DMZ avec pfSense
- **Active Directory** : domaine tssr.lan, gestion centralisée des utilisateurs, groupes (méthodologie AGDLP) et unités d'organisation(OU)
- **DNS et DHCP** : résolution de noms et attribution automatique d'adresses IP
- **GPO** : 7 stratégies de groupe pour la sécurité et la configuration des postes et utilisateurs (4 Computer, 3 User)
- **WSUS** : gestion centralisée des mises à jour Windows
- **GLPI** : gestion de parc informatique et ticketing avec synchronisation Active Directory
- **VoIP** : téléphonie IP avec FreePBX et softphone 3CX
- **Messagerie** : serveur mail iRedMail avec client Thunderbird
- **Postes clients** : Windows 10 et Windows 11 intégrés au domaine

### Objectifs secondaires

En complément des objectifs principaux, les éléments suivants ont été déployés :

- **Dossiers partagés** : dossier individuel par utilisateur, mappage automatique du lecteur I: via GPO
- **Redondance Active Directory** : deux contrôleurs de domaine supplémentaires (SRVWIN02 et SRVWIN03 en Windows Server Core) avec répartition des rôles FSMO
- **Restrictions horaires** : connexion autorisée de 7h30 à 20h00 du lundi au samedi pour les utilisateurs standard
- **NTP** : synchronisation horaire centralisée avec serveur autoritaire et GPO
- **Serveur web interne** : intranet Ekoloclast avec annuaire, organigramme et numéros de téléphone
- **Serveur web externe** : site vitrine en DMZ accessible depuis Internet via NAT pfSense
- **Synchronisation GLPI / Active Directory** : import automatique des utilisateurs et ordinateurs AD dans GLPI
- **GPO secondaires** : 7 stratégies supplémentaires (4 Computer, 3 User) pour WSUS, NTP, dossiers partagés, raccourci intranet et fond d'écran

## 2. Introduction

**Ekoloclast** est une start-up innovante fondée il y a moins de deux ans, installée à Paris dans le 8ème arrondissement.

Son ambition : révolutionner l'approche de l'écologie en introduisant des pratiques, produits et services écologiques novateurs qui bénéficient à la fois à l'environnement et aux individus.

Positionnée sur les marchés professionnels (B2B) et grand public (B2C), l'entreprise a récemment réussi une levée de fonds significative, lui ouvrant la voie vers une expansion prometteuse.

La société compte actuellement 183 collaborateurs répartis dans 10 départements : Communication, Direction Financière, Direction Générale, Direction Marketing, DSI, R&D, RH, Services Généraux, Service Juridique, et Ventes et Développement Commercial.

Des prestataires extérieurs interviennent ponctuellement ou à temps plein selon les besoins des différents services.

## 3. Choix techniques

### État actuel de l'entreprise

L'entreprise ne dispose actuellement d'aucune infrastructure réseau structurée :

- Aucun serveur ni matériel réseau dédié
- Tous les PC sont en workgroup avec des mots de passe faibles ou inexistants
- Connexion Internet via Wi-Fi fourni par une box FAI et des répéteurs
- Réseau actuel en 192.168.10.0/24
- Stockage des données en local sur les machines
- Sauvegardes ponctuelles sur un NAS, sans politique de rétention
- Messagerie hébergée en cloud (format prenom.nom@ekoloclast.fr)
- Téléphonie fixe et mobile non uniformisée
- Aucune sécurité d'identité

### Infrastructure cible

Domaine Active Directory : **tssr.lan**

Pour répondre aux besoins de l'entreprise, l'infrastructure suivante a été mise en place :

| Machine  | Rôle                          | OS                       | IP                                                     |
| -------- | ----------------------------- | ------------------------ | ------------------------------------------------------ |
| FW01     | Pare-feu (WAN/LAN/DMZ)        | pfSense CE 2.7.2         | WAN : DHCP, LAN : 192.168.10.254, DMZ : 192.168.30.254 |
| SRVWIN01 | AD DS + DNS + DHCP + GPO      | Windows Server 2022 GUI  | 192.168.10.5                                           |
| SRVWIN02 | Redondance AD DS + DNS (Core) | Windows Server 2022 Core | 192.168.10.10                                          |
| SRVWIN03 | Redondance AD DS + DNS (Core) | Windows Server 2022 Core | 192.168.10.15                                          |
| SRVWIN04 | WSUS                          | Windows Server 2022 GUI  | 192.168.10.20                                          |
| GLPI01   | Gestion de parc + Ticketing   | Debian 13 CLI            | 192.168.10.25                                          |
| IPBX01   | Téléphonie VoIP (FreePBX)     | Debian 12 CLI            | 192.168.10.30                                          |
| SRVLX01  | Messagerie (iRedMail)         | Debian 12 CLI            | 192.168.10.35                                          |
| SRVLX02  | Serveur web interne (LAN)     | Debian 12 CLI            | 192.168.10.40                                          |
| SRVLX03  | Serveur web externe (DMZ)     | Debian 12 CLI            | 192.168.30.5                                           |
| CLIWIN01 | Client                        | Windows 10               | DHCP                                                   |
| CLIWIN02 | Client                        | Windows 11               | DHCP                                                   |

Le plan d'adressage complet est disponible ici : [Plan_adressage_Ekoloclast.xlsx](Ressources/Plan_adressage_Ekoloclast.xlsx)

**Schéma réseau :** [Schema_Reseau_Ekoloclast.png](Ressources/Schema_Reseau_Ekoloclast.png)

### Hyperviseur

Toutes les machines virtuelles sont déployées sur **VirtualBox** sur infrastructure personnelle.

## 4. Difficultés rencontrées

### Active Directory et GPO

- **Méthodologie de création des GPO** : les GPO créées directement depuis une OU ne suivent pas les bonnes pratiques. La méthode correcte consiste à créer la GPO dans le conteneur « Group Policy Objects » de la console GPMC, puis à la lier à l'OU cible dans un second temps.

- **Correctif MS16-072 et filtrage de sécurité** : après application de ce correctif de sécurité Microsoft, les GPO de type Computer ne s'appliquaient plus. Le groupe « Authenticated Users » ayant été retiré du filtrage de sécurité, les machines n'avaient plus la permission de lire la GPO. Il est nécessaire d'ajouter le groupe « Domain Computers » avec la permission de lecture (GpoRead) sur chaque GPO Computer.

- **Icônes de raccourci non affichées sur chemin UNC** : les icônes personnalisées stockées sur un partage réseau (\\tssr.lan\NETLOGON\) ne s'affichaient pas sur les postes clients. Windows bloque par défaut le chargement d'icônes depuis un chemin réseau distant.

- **Action « Create » vs « Update » pour les raccourcis GPO** : après suppression manuelle d'un raccourci déployé par GPO avec l'action « Create », le raccourci ne se recrée pas. L'action « Create » ne traite le raccourci qu'une seule fois.

- **Désynchronisation horaire entre contrôleurs de domaine** : après extinction prolongée des DC de redondance (SRVWIN02 et SRVWIN03), la réplication Active Directory échouait avec l'erreur 1398 (différence de date/heure). Le protocole Kerberos refuse les échanges si l'écart dépasse 5 minutes.

- **Horloge client non synchronisée** : la commande gpupdate échouait sur les clients avec le message « l'horloge de cet ordinateur n'est pas synchronisée avec celle du contrôleur de domaine », empêchant l'application des stratégies Computer.

### GLPI sur Debian

- **Incompatibilité PHP 8.4 avec GLPI** : Debian 13 (Trixie) embarque PHP 8.4. GLPI 10.0.17 ne supporte pas cette version de PHP, provoquant des erreurs à l'installation.

- **Fichier resolv.conf écrasé au redémarrage** : sur les serveurs Debian configurés avec une adresse IP statique et un DNS spécifique (SRVWIN01), le fichier /etc/resolv.conf était systématiquement réinitialisé à chaque redémarrage, provoquant une perte de résolution DNS Active Directory.

- **mysql_secure_installation inexistant sur MariaDB 11.8** : sur Debian 13 avec MariaDB 11.8, la commande mysql_secure_installation n'existe plus. La sécurisation de la base de données (suppression des utilisateurs anonymes, désactivation de l'accès root distant, suppression de la base de test) ne peut pas se faire via cette commande.

- **Module php-imap manquant** : GLPI nécessite le module php-imap qui n'est pas installé par défaut sur Debian 13 avec PHP 8.3 via le dépôt Sury, provoquant des erreurs lors de l'utilisation des fonctionnalités de collecte de mails.

### iRedMail et messagerie

- **Certificat auto-signé et Thunderbird** : lors de la configuration de Thunderbird en tant que client de messagerie, le certificat SSL auto-signé généré par iRedMail n'est pas reconnu. Les versions récentes de Thunderbird (102+) ne proposent plus automatiquement d'accepter les certificats auto-signés pour les connexions IMAP et SMTP. L'import manuel du certificat et l'ajout d'exceptions de sécurité sont nécessaires.

### SRVLX01 — Positionnement réseau

- **Migration DMZ vers LAN** : le serveur SRVLX01 (iRedMail) était initialement prévu en DMZ (192.168.30.5). Or, iRedMail nécessite la résolution DNS Active Directory (tssr.lan) pour l'authentification des comptes. La résolution DNS inter-zones (DMZ → LAN) impliquait une règle pare-feu supplémentaire et introduisait une dépendance critique. Le serveur a dû être repositionné sur le LAN.

### FreePBX et VoIP

- **Sources APT obsolètes après installation** : l'installation de FreePBX modifie le fichier sources.list de Debian avec des dépôts legacy qui ne sont plus maintenus, empêchant toute mise à jour système via apt update.

- **Compatibilité softphone 3CX** : les versions récentes du client 3CX nécessitent un système 3CX complet et ne fonctionnent plus comme softphone SIP standalone. Il est nécessaire d'utiliser la version legacy (3CXPhone6.msi) qui reste compatible avec n'importe quel serveur SIP.

### WSUS

- **Délai de détection par défaut de 22 heures** : après configuration du rôle WSUS et application de la GPO sur les clients, ceux-ci n'apparaissent pas immédiatement dans la console WSUS. Le cycle de détection par défaut est de 22 heures.

- **Commande wuauclt obsolète** : la commande historique wuauclt /detectnow est dépréciée sur Windows 10/11 et n'a plus d'effet. Le remplacement par UsoClient.exe n'est pas documenté de manière évidente par Microsoft.

## 5. Solutions trouvées

### Active Directory et GPO

- **Méthodologie GPO** : adoption d'un workflow en 6 étapes conforme aux bonnes pratiques : (1) créer la GPO dans le conteneur « Group Policy Objects », (2) configurer les paramètres, (3) ajuster le filtrage de sécurité vers un groupe spécifique, (4) ajouter « Domain Computers » en lecture dans l'onglet Délégation, (5) lier la GPO à l'OU cible, (6) tester avec gpresult /r sur un poste client.

- **Correctif MS16-072** : ajout systématique du groupe « Domain Computers » avec la permission « Read » (GpoRead) dans l'onglet « Délégation » de chaque GPO Computer. Cette opération est réalisée via la console GPMC, onglet Délégation → Avancé.

- **Icônes UNC** : création et application d'une GPO Computer (COMPUTER-Explorer-RemoteIcons) activant le paramètre « Allow the use of remote paths in file shortcut icons » dans Computer Configuration → Administrative Templates → Windows Components → File Explorer. Un redémarrage du poste client est nécessaire après application.

- **Action raccourci GPO** : utilisation de l'action « Update » au lieu de « Create » pour les raccourcis déployés par GPO Preferences. L'action « Update » crée le raccourci s'il n'existe pas et le met à jour s'il existe déjà, y compris après une suppression manuelle.

- **Désynchronisation horaire DC** : correction manuelle de l'heure sur les DC de redondance via PowerShell (Set-Date), puis synchronisation forcée de la réplication avec la commande repadmin /syncall /AdeP depuis le DC principal.

- **Horloge client** : resynchronisation de l'horloge du client avec la commande w32tm /resync /force en invite de commandes administrateur.

### GLPI sur Debian

- **Incompatibilité PHP** : installation du dépôt **Sury** sur Debian 13 pour obtenir PHP 8.3, version compatible avec GLPI 10.0.17.

- **Fichier resolv.conf** : verrouillage du fichier avec la commande chattr +i /etc/resolv.conf pour empêcher sa modification au redémarrage. Le DNS pointe vers SRVWIN01 (192.168.10.5).

- **mysql_secure_installation** : sécurisation directe de MariaDB via des commandes SQL (ALTER USER, DELETE FROM mysql.user, DROP DATABASE test, FLUSH PRIVILEGES) en remplacement de la commande mysql_secure_installation absente sur MariaDB 11.8.

- **Module php-imap** : installation manuelle du paquet php8.3-imap depuis le dépôt Sury, suivie d'un redémarrage d'Apache (systemctl restart apache2).

### iRedMail et messagerie

- **Certificat auto-signé** : récupération du certificat depuis le serveur (cat /etc/ssl/certs/iRedMail.crt), import dans Thunderbird via Settings → Privacy & Security → Manage Certificates → Authorities, déblocage des ports dans le Config Editor (network.security.ports.banned.override : 993,587,465,25), et ajout des exceptions de sécurité pour les ports 465 et 993.

### SRVLX01 ( Positionnement réseau)

- **Migration vers le LAN** : repositionnement de SRVLX01 sur le réseau LAN avec l'adresse **192.168.10.35**. Cette solution simplifie la résolution DNS (accès direct à SRVWIN01) et évite la création de règles pare-feu DMZ → LAN pour le port 53.

### FreePBX et VoIP

- **Sources APT** : correction manuelle du fichier /etc/apt/sources.list pour rétablir les dépôts Debian officiels après l'installation de FreePBX.

- **Softphone** : utilisation de la version legacy **3CX Phone** (3CXPhone6.msi), un softphone standalone compatible avec n'importe quel serveur SIP (FreePBX, Asterisk) sans nécessiter un système 3CX complet.

### WSUS

- **Délai de détection** : exécution manuelle de la commande UsoClient.exe StartScan sur les postes clients pour forcer la détection immédiate sans attendre le cycle de 22 heures.

- **Commande de remplacement** : utilisation de UsoClient.exe en remplacement de wuauclt pour toutes les opérations de mise à jour sur Windows 10/11 (StartScan, StartDownload, StartInstall).

## 6. Améliorations possibles

### Analyse de capacité et évolutivité

#### Dimensionnement actuel

L'infrastructure déployée utilise un plan d'adressage en /24 (254 adresses disponibles) pour le réseau LAN. Ce choix est adapté au contexte du projet de formation avec un nombre limité de machines virtuelles.

#### Limites identifiées pour une mise en production

Dans le contexte réel d'Ekoloclast (183 collaborateurs), le dimensionnement actuel présente des contraintes :

| Élément            | Quantité estimée |
| ------------------ | ---------------- |
| PC portables       | 183              |
| Téléphones IP      | 183              |
| Serveurs           | 10               |
| Imprimantes réseau | 10               |
| Bornes WiFi        | 15               |
| **Total estimé**   | 400 IP           |

Un réseau /24 (254 adresses) serait saturé dès le déploiement complet.

#### Améliorations recommandées

| Amélioration         | Description                             | Bénéfice                      |
| -------------------- | --------------------------------------- | ----------------------------- |
| Segmentation VLANs   | VLAN Users / VLAN Servers / VLAN VoIP   | Sécurité et QoS               |
| pfSense HA           | Cluster de deux pare-feu en failover    | Élimination du SPOF           |
| WDS/MDT              | Déploiement automatisé des postes       | Gain de temps (183 PC)        |
| VPN OpenVPN          | Accès distant sécurisé pour les nomades | Télétravail / Mobilité        |
| WiFi WPA2-Enterprise | Authentification via NPS/RADIUS et AD   | Sécurité WiFi professionnelle |

#### Architecture VLAN cible (évolution future)

| VLAN    | Nom          | Réseau          | Usage                   |
| ------- | ------------ | --------------- | ----------------------- |
| VLAN 10 | Utilisateurs | 192.168.10.0/24 | PC portables            |
| VLAN 20 | Serveurs     | 192.168.20.0/24 | Infrastructure serveurs |
| VLAN 30 | VoIP         | 192.168.30.0/24 | Téléphones IP           |
| VLAN 40 | DMZ          | 192.168.40.0/24 | Services exposés        |

Cette segmentation permettrait d'isoler les flux, d'appliquer des règles de pare-feu inter-VLANs et de prioriser le trafic voix (QoS). Cette évolution impliquerait un re-adressage de la DMZ actuelle (192.168.30.0/24 → 192.168.40.0/24) pour libérer le sous-réseau 30 au profit du VLAN VoIP.