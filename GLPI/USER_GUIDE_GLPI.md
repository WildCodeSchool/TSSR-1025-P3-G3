
## Utilisation de base

### Accéder à GLPI

1. Ouvrir un navigateur web
2. Aller à : http://192.168.10.25/glpi
3. Entrer les identifiants :
   - **Identifiant** : glpi (ou votre compte)
   - **Mot de passe** : glpi (ou votre mot de passe)
4. Cliquer sur **Se connecter**

---

### Créer un ticket d'assistance

1. Se connecter à GLPI
2. Aller dans **Assistance** → **Créer un ticket**
3. Remplir le formulaire :
   - **Type** : Incident ou Demande
   - **Catégorie** : sélectionner la catégorie appropriée
   - **Titre** : description courte du problème
   - **Description** : détails du problème
4. Cliquer sur **Soumettre le ticket**
5. Le ticket est créé et un numéro est attribué

---

### Suivre ses tickets

1. Se connecter à GLPI
2. Aller dans **Assistance** → **Tickets**
3. La liste de vos tickets s'affiche
4. Cliquer sur un ticket pour voir les détails

---

### Ajouter un suivi à un ticket

1. Ouvrir le ticket concerné
2. Aller dans l'onglet **Suivis**
3. Cliquer sur **Ajouter un nouveau suivi**
4. Entrer le message
5. Cliquer sur **Ajouter**

---

## Utilisation avancée

### Consulter le parc informatique (Admin)

1. Se connecter avec un compte admin
2. Aller dans **Parc** → **Ordinateurs**
3. Cliquer sur un ordinateur pour voir les détails

---

### Installation de GLPI Agent (inventaire automatique)

La synchronisation Active Directory importe uniquement les **utilisateurs** dans GLPI. Pour importer automatiquement les **ordinateurs** du domaine (nom, OS, RAM, CPU, disques, logiciels, réseau), il faut installer **GLPI Agent** sur chaque poste client et serveur.

**Version utilisée :** GLPI Agent 1.16 (compatible GLPI 10.0.17)

**Source :** https://github.com/glpi-project/glpi-agent/releases/tag/1.16

#### Prérequis sur GLPI01

Avant d'installer l'agent, vérifier que l'inventaire natif est activé sur le serveur GLPI :

1. Se connecter à http://192.168.10.25/glpi en tant que **glpi**
2. Aller dans **Administration** → **Inventaire**
3. Vérifier que **Activer l'inventaire** est sur **Oui**
4. Si ce n'est pas le cas, activer l'option et cliquer sur **Enregistrer**

#### Installation sur les machines Windows (CLIWIN01, CLIWIN02, SRVWIN01, SRVWIN04)

**Fichier à télécharger :** GLPI-Agent-1.16-x64.msi (identique pour Windows 10, Windows 11 et Windows Server)

1. Se connecter sur la machine avec un compte du domaine
2. Ouvrir un navigateur et aller sur : https://github.com/glpi-project/glpi-agent/releases/tag/1.16
3. Télécharger le fichier **GLPI-Agent-1.16-x64.msi**
4. Double-cliquer sur le fichier téléchargé pour lancer l'installation
5. Sur l'écran d'accueil, cliquer sur **Next**
6. Accepter la licence → **Next**
7. Choisir le type d'installation : **Typical** → **Next**
8. Dans l'écran **Choose Targets**, remplir le champ **Remote Targets** avec :

       http://192.168.10.25/glpi

9. Laisser le champ **Local Target** vide
10. Cliquer sur **Next** 
11. Cliquer sur **Install**
12. La fenêtre de contrôle UAC s'affiche : entrer les identifiants administrateur (Administrator / Azerty1*) et cliquer sur **Oui**
13. Attendre la fin de l'installation
14. Cliquer sur **Finish**

**Effectuer la même installation sur chaque machine Windows** (CLIWIN02, SRVWIN01, SRVWIN04).

#### Installation à distance sur les serveurs Core (SRVWIN02, SRVWIN03)

SRVWIN02 et SRVWIN03 sont en **Server Core** (pas de navigateur ni d'interface graphique). L'installation se fait **à distance depuis SRVWIN01** via PowerShell.

##### Étape 1 — Créer le dossier temporaire et copier le MSI

Le fichier **GLPI-Agent-1.16-x64.msi** doit déjà être téléchargé sur SRVWIN01 (dans `C:\Users\Administrator\Downloads\`).

```powershell
# Créer le dossier Temp sur les serveurs Core
Invoke-Command -ComputerName SRVWIN02 -ScriptBlock { New-Item -Path "C:\Temp" -ItemType Directory -Force }
Invoke-Command -ComputerName SRVWIN03 -ScriptBlock { New-Item -Path "C:\Temp" -ItemType Directory -Force }

# Copier le MSI via le partage administratif C$
Copy-Item -Path "C:\Users\Administrator\Downloads\GLPI-Agent-1.16-x64.msi" -Destination "\\SRVWIN02\C$\Temp\"
Copy-Item -Path "C:\Users\Administrator\Downloads\GLPI-Agent-1.16-x64.msi" -Destination "\\SRVWIN03\C$\Temp\"
```

##### Étape 2 — Installer GLPI Agent à distance

```powershell
# Installer sur SRVWIN02
Invoke-Command -ComputerName SRVWIN02 -ScriptBlock {
    Start-Process msiexec.exe -ArgumentList '/i "C:\Temp\GLPI-Agent-1.16-x64.msi" /quiet SERVER=http://192.168.10.25/glpi' -Wait
}

# Installer sur SRVWIN03
Invoke-Command -ComputerName SRVWIN03 -ScriptBlock {
    Start-Process msiexec.exe -ArgumentList '/i "C:\Temp\GLPI-Agent-1.16-x64.msi" /quiet SERVER=http://192.168.10.25/glpi' -Wait
}
```

##### Étape 3 — Vérifier et forcer l'inventaire

```powershell
# Vérifier le service sur SRVWIN02
Invoke-Command -ComputerName SRVWIN02 -ScriptBlock {
    Get-Service -Name "GLPI-Agent" | Select-Object Name, Status
}

# Vérifier le service sur SRVWIN03
Invoke-Command -ComputerName SRVWIN03 -ScriptBlock {
    Get-Service -Name "GLPI-Agent" | Select-Object Name, Status
}

# Forcer l'inventaire sur SRVWIN02
Invoke-Command -ComputerName SRVWIN02 -ScriptBlock {
    Invoke-WebRequest -Uri "http://127.0.0.1:62354/now" -UseBasicParsing
}

# Forcer l'inventaire sur SRVWIN03
Invoke-Command -ComputerName SRVWIN03 -ScriptBlock {
    Invoke-WebRequest -Uri "http://127.0.0.1:62354/now" -UseBasicParsing
}
```

Résultat attendu : `Status = Running` et la page retourne **OK**.

##### Étape 4 — Nettoyage

```powershell
Invoke-Command -ComputerName SRVWIN02 -ScriptBlock { Remove-Item "C:\Temp\GLPI-Agent-1.16-x64.msi" -Force }
Invoke-Command -ComputerName SRVWIN03 -ScriptBlock { Remove-Item "C:\Temp\GLPI-Agent-1.16-x64.msi" -Force }
```

#### Installation sur les serveurs Debian (SRVLX01, IPBX01)

**Note** : GLPI01 n'a pas besoin de l'agent, c'est le serveur GLPI lui-même.

1. Se connecter en **root** sur le serveur Debian

2. Télécharger l'installateur :

       wget https://github.com/glpi-project/glpi-agent/releases/download/1.16/glpi-agent-1.16-linux-installer.pl

3. Lancer l'installation :

       perl glpi-agent-1.16-linux-installer.pl --server=http://192.168.10.25/glpi

4. Forcer l'inventaire :

       glpi-agent --force

**Effectuer la même installation sur IPBX01.**

#### Ajout manuel de FW01 (pfSense)

FW01 fonctionne sous pfSense (FreeBSD). GLPI Agent n'est pas compatible avec pfSense, l'équipement doit être ajouté manuellement :

1. Se connecter à http://192.168.10.25/glpi en tant que **glpi**
2. Aller dans **Parc** → **Équipements réseau**
3. Cliquer sur **+**
4. Remplir les informations :
   - **Nom** : FW01
   - **Type** : Pare-feu
   - **Fabricant** : Netgate
   - **Système d'exploitation** : pfSense CE 2.7.2
   - **IP** : 192.168.10.254 (LAN)
5. Cliquer sur **Ajouter**

#### Forcer un inventaire immédiat

Après l'installation, l'agent envoie automatiquement un inventaire à GLPI toutes les **24 heures**. Pour forcer l'inventaire immédiatement :

**Sur les machines Windows (GUI)** : ouvrir un navigateur et aller à http://127.0.0.1:62354/now → la page affiche **OK**

**Sur les serveurs Core (SRVWIN02, SRVWIN03)** : depuis SRVWIN01 en PowerShell :

```powershell
Invoke-Command -ComputerName SRVWIN02 -ScriptBlock { Invoke-WebRequest -Uri "http://127.0.0.1:62354/now" -UseBasicParsing }
Invoke-Command -ComputerName SRVWIN03 -ScriptBlock { Invoke-WebRequest -Uri "http://127.0.0.1:62354/now" -UseBasicParsing }
```

**Sur les serveurs Debian** (en root) :

    glpi-agent --force

#### Vérification dans GLPI

1. Se connecter à http://192.168.10.25/glpi en tant que **glpi**
2. Aller dans **Parc** → **Ordinateurs**
3. Les machines suivantes doivent apparaître dans la liste :

  - CLIWIN01 
  - CLIWIN02

4. Cliquer sur un ordinateur pour voir les détails de l'inventaire (OS, RAM, CPU, disques, logiciels installés, configuration réseau)

---

### Ajouter un équipement au parc manuellement (Admin)

1. Aller dans **Parc** → **Ordinateurs**
2. Cliquer sur **+**
3. Remplir les informations :
   - Nom
   - Numéro de série
   - Système d'exploitation
   - Utilisateur
4. Cliquer sur **Ajouter**

---

### Traiter un ticket (Admin/Technicien)

1. Aller dans **Assistance** → **Tickets**
2. Cliquer sur un ticket
3. Cliquer sur **Prendre en charge**
4. Ajouter un suivi ou une solution
5. Cliquer sur **Résoudre** quand terminé

---

### Clôturer un ticket (Admin/Technicien)

1. Ouvrir le ticket résolu
2. Cliquer sur **Clôturer**

---

## FAQ

### Je ne trouve pas mon ticket
- Vérifier les filtres appliqués
- Aller dans **Assistance** → **Tickets** et cliquer sur "Tous"
- Utiliser la recherche

### Je n'arrive pas à me connecter
- Vérifier l'identifiant et le mot de passe
- Contacter l'administrateur si le problème persiste

### Comment changer mon mot de passe
1. Cliquer sur votre nom en haut à droite
2. Aller dans **Mon compte**
3. Changer le mot de passe dans l'onglet **Principal**

### Les ordinateurs n'apparaissent pas dans le parc
- Vérifier que GLPI Agent est installé sur le poste client
- Vérifier que l'inventaire natif est activé dans GLPI (Administration → Inventaire)
- Forcer un inventaire : ouvrir http://127.0.0.1:62354/now sur le poste client (Windows) ou glpi-agent --force (Debian)
- Pour les serveurs Core : utiliser Invoke-Command depuis SRVWIN01 (voir section Installation à distance)
- Vérifier que le poste client peut joindre GLPI01 (192.168.10.25) sur le réseau