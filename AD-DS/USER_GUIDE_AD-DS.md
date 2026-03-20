## Utilisation de base

### Se connecter au domaine (depuis un client)

1. Ouvrir **Paramètres** → **Système** → **Informations système**
2. Cliquer sur **Paramètres avancés du système**
3. Aller dans l'onglet **Nom de l'ordinateur**
4. Cliquer sur **Modifier**
5. Sélectionner **Domaine**
6. Entrer : tssr.lan
7. Cliquer sur **OK**
8. Entrer les identifiants :
   - **Nom d'utilisateur** : Administrator
   - **Mot de passe** : Azerty1*
9. Cliquer sur **OK**

10. Message de bienvenue dans le domaine
11. Redémarrer l'ordinateur

---

### Se connecter avec un compte du domaine

1. Sur l'écran de connexion Windows
2. Cliquer sur **Autre utilisateur**
3. Entrer le nom d'utilisateur : petra.rossi@tssr.lan ou TSSR\petra.rossi
4. Entrer le mot de passe : Azerty1*
5. Appuyer sur **Entrée**

---

### Vérifier l'appartenance au domaine

1. Ouvrir **Invite de commandes**
2. Taper : whoami
3. Le résultat doit afficher : tssr\petra.rossi
4. Taper : gpresult /r
5. Vérifier les GPO appliquées

---

## Utilisation avancée

### Réinitialiser le mot de passe d'un utilisateur (Admin)

Sur **SRVWIN01** (interface en anglais) :

1. Ouvrir **Active Directory Users and Computers**
2. Trouver l'utilisateur dans son OU
3. Clic droit → **Reset Password**
4. Entrer le nouveau mot de passe
5. Cocher **User must change password at next logon**
6. Cliquer sur **OK**



---

### Déverrouiller un compte utilisateur (Admin)

Sur **SRVWIN01** (interface en anglais) :

1. Ouvrir **Active Directory Users and Computers**
2. Trouver l'utilisateur dans son OU
3. Double-clic sur l'utilisateur
4. Aller dans l'onglet **Account**
5. Décocher **Unlock account**
6. Cliquer sur **OK**


---

### Ajouter un utilisateur à un groupe (Admin)

Sur **SRVWIN01** (interface en anglais) :

1. Ouvrir **Active Directory Users and Computers**
2. Trouver l'utilisateur dans son OU
3. Double-clic sur l'utilisateur
4. Aller dans l'onglet **Member Of**
5. Cliquer sur **Add**

6. Entrer le nom du groupe
7. Cliquer sur **Check Names**
8. Cliquer sur **OK**



**Note :** L'utilisateur doit être ajouté au groupe **Grp_** (Global) de son département. La chaîne AGDLP (Grp_ → DL_ → GPO) fait le reste automatiquement.

---

### Vérifier les GPO appliquées sur un poste

1. Sur le poste client, ouvrir **Invite de commandes** en administrateur
2. Taper : gpresult /h rapport.html
3. Ouvrir le fichier rapport.html généré

---

### Forcer la mise à jour des GPO

1. Sur le poste client, ouvrir **Invite de commandes** en administrateur
2. Taper : gpupdate /force
3. Attendre le message de confirmation


---

## FAQ

### Je n'arrive pas à me connecter au domaine
- Vérifier que le câble réseau est branché
- Vérifier que l'adresse IP est correcte (DHCP ou manuelle)
- Vérifier que le DNS pointe vers SRVWIN01 (192.168.10.5)
- Tester avec ping srvwin01.tssr.lan

### Mon compte est verrouillé
- Contacter l'administrateur pour déverrouiller le compte
- Attendre 10 minutes (déverrouillage automatique)

### Les GPO ne s'appliquent pas
- Exécuter gpupdate /force
- Redémarrer l'ordinateur
- Vérifier avec gpresult /r que les GPO sont bien listées
- Pour les GPO utilisateur : vérifier que le compte est bien membre du groupe Grp_ de son département

### Erreur "La relation d'approbation entre cette station de travail et le domaine principal a échoué"
- Contacter l'administrateur
- Solution admin : sortir le PC du domaine puis le réintégrer