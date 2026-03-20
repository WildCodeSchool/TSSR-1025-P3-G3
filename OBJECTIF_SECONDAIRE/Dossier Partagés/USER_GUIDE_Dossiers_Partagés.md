## Utilisation de base

### Accéder à son dossier personnel

1. Ouvrir **Ce PC** (This PC) depuis le bureau ou l'explorateur de fichiers
2. Le lecteur **I:** nommé **Dossier Personnel** apparaît automatiquement
3. Double-cliquer sur **I:** pour ouvrir le dossier

---

### Enregistrer un fichier dans son dossier personnel

1. Ouvrir l'application souhaitée (Word, Excel, etc.)
2. Cliquer sur **Fichier** → **Enregistrer sous**
3. Sélectionner **I:** (Dossier Personnel)
4. Nommer le fichier et cliquer sur **Enregistrer**

---

### À quoi sert le lecteur I: ?

Le lecteur I: est un espace de stockage **personnel et privé** hébergé sur le serveur de l'entreprise.

- Seul vous et les administrateurs y avez accès
- Les autres utilisateurs ne peuvent pas voir vos fichiers
- Les fichiers sont stockés sur le serveur, pas sur votre PC
- En cas de panne de votre PC, vos fichiers restent accessibles depuis un autre poste

---

## Utilisation avancée

### Accéder à son dossier depuis un autre poste

Si vous changez de PC ou utilisez temporairement un autre poste du domaine :

1. Se connecter avec votre compte habituel (ex: kian.kowalski)
2. Le lecteur **I:** apparaît automatiquement
3. Tous vos fichiers sont accessibles

---

### Accéder manuellement au dossier si le lecteur I: n'apparaît pas

1. Ouvrir **l'Explorateur de fichiers**
2. Dans la barre d'adresse, taper : \\\\SRVWIN01\Individuels$\votre.login
3. Remplacer **votre.login** par votre nom d'utilisateur (ex: kian.kowalski)
4. Appuyer sur **Entrée**

---

### Forcer l'apparition du lecteur I:

Si le lecteur I: n'apparaît pas après connexion :

1. Ouvrir **Invite de commandes** (taper **cmd** dans la barre de recherche)
2. Taper la commande suivante :

```cmd
gpupdate /force
```

3. Fermer la session (Démarrer → icône utilisateur → **Se déconnecter**)
4. Se reconnecter avec votre compte
5. Le lecteur I: devrait apparaître dans Ce PC

---

## FAQ

### Le lecteur I: n'apparaît pas
- Fermer la session et se reconnecter
- Si toujours absent, ouvrir **Invite de commandes** et taper : gpupdate /force puis se reconnecter
- Si le problème persiste, contacter l'administrateur

### "Accès refusé" en ouvrant le lecteur I:
- Vérifier que vous êtes connecté avec votre propre compte (pas celui d'un collègue)
- Contacter l'administrateur pour vérifier les permissions de votre dossier

### J'ai supprimé un fichier par erreur
- Le fichier n'est pas dans la corbeille de votre PC (il est sur le serveur)
- Contacter l'administrateur pour une éventuelle restauration

### Puis-je partager mon dossier avec un collègue ?
- Non, le dossier personnel est strictement individuel
- Pour partager des fichiers entre collègues, utiliser un autre moyen (mail, dossier de service partagé si existant)