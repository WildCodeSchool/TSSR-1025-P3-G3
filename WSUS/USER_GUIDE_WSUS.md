# USER GUIDE WSUS

## Utilisation de base

### Vérifier les mises à jour sur un poste client

1. Ouvrir **Paramètres** → **Windows Update**
2. Cliquer sur **Rechercher des mises à jour**
3. Les mises à jour approuvées par l'administrateur s'affichent

---

### Installer les mises à jour

1. Dans **Windows Update**
2. Cliquer sur **Télécharger et installer**
3. Attendre la fin du téléchargement et de l'installation
4. Redémarrer si demandé

---

### Forcer la recherche de mises à jour

1. Ouvrir **PowerShell** en administrateur
2. Taper : UsoClient.exe StartScan
3. Retourner dans **Windows Update** et actualiser

**Note** : La commande wuauclt /detectnow est dépréciée sur Windows 10 et Windows 11 et n'a aucun effet. Utiliser UsoClient.exe StartScan à la place.

---

## Utilisation avancée

### Consulter l'historique des mises à jour

1. Ouvrir **Paramètres** → **Windows Update**
2. Cliquer sur **Historique des mises à jour**
3. Voir la liste des mises à jour installées

---

### Accéder à la console WSUS (Admin)

1. Sur SRVWIN04
2. Ouvrir **Gestionnaire de serveur**
3. Cliquer sur **Outils** → **Windows Server Update Services**

---

### Approuver des mises à jour (Admin)

1. Dans la console WSUS
2. Aller dans **Mises à jour** → **Toutes les mises à jour**
3. Filtrer par **Approbation : Non approuvées**
4. Sélectionner les mises à jour
5. Clic droit → **Approuver**
6. Sélectionner le groupe et l'action
7. Cliquer sur **OK**


---

### Générer un rapport (Admin)

1. Dans la console WSUS
2. Aller dans **Rapports**
3. Sélectionner le type de rapport souhaité
4. Cliquer sur **Exécuter le rapport**

---

## FAQ

### Les mises à jour ne s'affichent pas
- Vérifier la connexion au serveur WSUS
- Exécuter UsoClient.exe StartScan en PowerShell administrateur
- Attendre quelques heures (synchronisation périodique)
- Contacter l'administrateur

### Erreur lors de l'installation d'une mise à jour
- Redémarrer le poste et réessayer
- Vérifier l'espace disque disponible
- Consulter l'historique pour voir le code d'erreur
- Contacter l'administrateur

### Le poste demande constamment de redémarrer
- Redémarrer le poste pour finaliser l'installation
- Si le problème persiste, contacter l'administrateur