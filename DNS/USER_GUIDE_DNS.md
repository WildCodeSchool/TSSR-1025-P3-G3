
## Utilisation de base

### Vérifier la configuration DNS d'un poste

1. Ouvrir **Command Prompt**
2. Taper : ipconfig /all
3. Vérifier la ligne **DNS Servers** : doit afficher 192.168.10.5



---

### Tester la résolution DNS

1. Ouvrir **Command Prompt**
2. Taper : nslookup nom_machine.tssr.lan
3. Exemple : nslookup srvwin01.tssr.lan

---

### Tester la connexion Internet

1. Ouvrir **Command Prompt**
2. Taper : nslookup google.com
3. Une adresse IP doit être retournée

---

## Utilisation avancée

### Vider le cache DNS local

1. Ouvrir **Command Prompt** en administrateur
2. Taper : ipconfig /flushdns
3. Message : "Successfully flushed the DNS Resolver Cache"

---

### Afficher le cache DNS local

1. Ouvrir **Command Prompt**
2. Taper : ipconfig /displaydns
3. Liste des entrées DNS en cache

---

### Tester avec un serveur DNS spécifique

1. Ouvrir **Command Prompt**
2. Taper : nslookup
3. Taper : server 192.168.10.5
4. Taper le nom à résoudre : srvwin01.tssr.lan

---

## FAQ

### "Impossible de trouver l'hôte" ou "Can't find host"
- Vérifier que le DNS est configuré sur 192.168.10.5
- Vérifier la connexion réseau avec ping 192.168.10.5
- Vider le cache DNS avec ipconfig /flushdns

### La résolution fonctionne mais pas Internet
- Vérifier les forwarders DNS sur le serveur
- Vérifier les règles pare-feu sur FW01
- Tester avec ping 8.8.8.8 (test sans DNS)

### Délai de résolution très long
- Vérifier que le serveur DNS est accessible
- Vérifier qu'il n'y a pas de serveur DNS secondaire incorrect