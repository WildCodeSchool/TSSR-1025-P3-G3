# USER GUIDE Restrictions d'utilisation

## Vérifier les horaires d'un utilisateur

1. Sur **SRVWIN01**, ouvrir **Active Directory Users and Computers** (dsa.msc)
2. Naviguer dans l'OU du département
3. Double-cliquer sur l'utilisateur → onglet **Account** → **Logon Hours...**
4. Les cases **bleues** = connexion autorisée, les cases **blanches** = connexion interdite

---

## Modifier les horaires d'un utilisateur

1. Dans **dsa.msc**, double-cliquer sur l'utilisateur
2. Onglet **Account** → **Logon Hours...**
3. Sélectionner les plages horaires à modifier
4. Cliquer sur **Logon Permitted** (bleu) ou **Logon Denied** (blanc)
5. **OK** → **Apply** → **OK**

---

## Remettre un utilisateur en accès 24h/24

1. Double-cliquer sur l'utilisateur → onglet **Account** → **Logon Hours...**
2. Sélectionner toutes les cases (cliquer sur le coin supérieur gauche de la grille)
3. Cliquer sur **Logon Permitted**
4. **OK** → **Apply** → **OK**

---

## Vérifier qu'un utilisateur est bien bloqué

Si un utilisateur tente de se connecter en dehors des horaires autorisés :

- **Sur le poste client** : message d'erreur à l'écran de connexion
- **Sur le DC** : vérifier dans l'Event Viewer → **Security** → Event ID **4625** (logon failure)

---

## Vérification rapide en PowerShell

Pour voir l'état des restrictions de tous les utilisateurs :

```powershell
Get-ADUser -Filter * -Properties logonHours | Select-Object Name, @{N='Restriction';E={if($null -eq $_.logonHours -or ($_.logonHours | Where-Object {$_ -ne 255}).Count -eq 0){"Aucune"}else{"Active"}}} | Format-Table -AutoSize
```

---

## Important — Arrêt et redémarrage du serveur

L'attribut logonHours est stocké dans la base Active Directory (NTDS.dit). Pour que les modifications soient conservées après un redémarrage :

- **Toujours** éteindre le serveur proprement (Démarrer → Shut down ou Restart-Computer)
- **En environnement virtualisé** (VirtualBox, Hyper-V) : ne jamais utiliser Power Off. Utiliser **Send Shutdown Signal (ACPI)** ou **Close → Send the shutdown signal**
- Un arrêt brutal (coupure de courant, Power Off) peut entraîner la perte des dernières modifications AD non sauvegardées sur disque

---

## Utilisation avancée — Script interactif

Le script **Set-LogonHoursRestriction.ps1** est un outil interactif universel qui fonctionne sur n'importe quel domaine Active Directory.

### Fonctionnalités

- **100% interactif** : aucun paramètre à taper, menus avec navigation fluide
- **Scan automatique** : analyse la forêt, le domaine, les OUs et tous les utilisateurs
- **Plages horaires personnalisables** : choix de l'heure de début, fin et jours
- **Correction automatique UTC** : les horaires AD sont stockés en UTC, le script corrige le décalage fuseau
- **3 modes de ciblage** : tous les standards, OUs spécifiques, ou tous (admins inclus)
- **Consultation** : naviguer dans les OUs et voir le détail des utilisateurs
- **Rapport avant/après** : état de chaque utilisateur avant et après modification
- **Mode WhatIf** : simuler sans appliquer
- **Double tentative** en cas d'erreur (DN puis SamAccountName)
- **Logs + CSV** dans C:\Logs\LogonHours\
- **Retour** : touche [0] pour revenir à l'écran précédent depuis n'importe où

### Prérequis

- Copier le script **Set-LogonHoursRestriction.ps1** sur un DC ou un poste avec RSAT (ex : C:\Scripts\)
- Ouvrir **PowerShell en administrateur**

### Lancement

```powershell
cd C:\Scripts
.\Set-LogonHoursRestriction.ps1
```

### Navigation

Le script affiche un menu principal avec les informations du domaine :

```
============================================================
  GESTION DES HORAIRES DE CONNEXION AD
============================================================

  Date : 12/02/2026
  Heure : 14H30
  Foret : tssr.lan
  Domaine : tssr.lan
  DC : SRVWIN01
  Unites organisationnelles : 9

  [1] Appliquer des restrictions d'horaires
  [2] Retirer les restrictions (remettre 24h/24)
  [3] Consulter les OUs et utilisateurs
  [Q] Quitter
```

**[1] Appliquer des restrictions** :

1. Choisir la cible (tous les standards, OUs spécifiques, ou tous)
2. Si OUs spécifiques : sélectionner les numéros (ex: 1,3,5 ou 1-5 ou tout)
3. Voir la liste des utilisateurs sélectionnés → [C] continuer ou [0] retour
4. Définir l'heure de début et de fin (ex : 7 et 20 pour 7h-20h)
5. Choisir les jours (lun-ven, tous, ou personnalisé)
6. Récapitulatif → [O] appliquer, [W] simuler, [0] retour

**[2] Retirer les restrictions** :

1. Choisir la cible
2. Confirmation → les utilisateurs repassent en 24h/24

**[3] Consulter** :

1. Voir la liste des OUs avec nombre d'utilisateurs et restrictions
2. Entrer un numéro d'OU pour voir le détail de chaque utilisateur
3. [0] retour à la liste des OUs

### Résultat

Après application, le script affiche un résumé (utilisateurs traités, erreurs, durée) et propose :

- **[R]** Ouvrir le rapport CSV
- **[M]** Retour au menu principal (re-scan automatique pour voir les changements)
- **[Q]** Quitter

### Fichiers générés

Les fichiers sont créés dans **C:\Logs\LogonHours\** :

| Fichier | Description |
| ------- | ----------- |
| LogonHours_[date].log | Log détaillé de chaque action |
| LogonHours_Report_[date].csv | Rapport avec état avant/après par utilisateur |

Le rapport CSV contient :

| Colonne | Description |
| ------- | ----------- |
| Status | OK, WHATIF, ERREUR, OK (2e tentative) |
| OU | Unité organisationnelle |
| Name | Nom complet |
| SamAccountName | Identifiant AD |
| Enabled | Compte actif ou non |
| Restriction_Before | État avant modification |
| Restriction_After | État après modification |
| LogonHours_Before | Octets bruts avant |
| LogonHours_After | Octets bruts après |
| WhenProcessed | Date/heure du traitement |
| Error | Message d'erreur (si applicable) |

### Détail technique — Correction UTC

Les horaires de connexion AD sont stockés en UTC dans l'attribut logonHours (tableau de 21 octets). Le script détecte automatiquement le fuseau horaire du serveur et décale les bits en conséquence. Par exemple, si le DC est en UTC+1 (France), une restriction 7h-20h en heure locale sera stockée comme 6h-19h en UTC.

Cette correction est basée sur la méthode documentée par IT-Connect et utilise la commande Get-TimeZone pour calculer le décalage.