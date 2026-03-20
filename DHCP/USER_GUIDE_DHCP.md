# USER GUIDE DHCP

## Utilisation de base

### Vérifier la configuration IP automatique

1. Ouvrir **Command Prompt**
2. Taper : ipconfig /all
3. Vérifier :
   - **DHCP Enabled** : Yes
   - **IPv4 Address** : 192.168.10.1xx
   - **DHCP Server** : 192.168.10.5

---

### Renouveler l'adresse IP

1. Ouvrir **Command Prompt** en administrateur
2. Taper : ipconfig /release
3. Taper : ipconfig /renew


---

## Utilisation avancée

### Voir les baux actifs (Admin)

1. Sur SRVWIN01, ouvrir **Server Manager**
2. Cliquer sur **Tools** → **DHCP**
3. Développer **IPv4** → **Scope** → **Address Leases**

---

### Réserver une adresse IP pour une machine (Admin)

1. Dans la console DHCP
2. Clic droit sur **Reservations**
3. Cliquer sur **New Reservation**
4. Entrer :
   - **Reservation name** : Nom de la machine
   - **IP address** : IP à réserver
   - **MAC address** : Adresse MAC de la machine
5. Cliquer sur **Add**


---

### Trouver l'adresse MAC d'une machine

1. Ouvrir **Command Prompt**
2. Taper : ipconfig /all
3. Noter la **Physical Address** (format XX-XX-XX-XX-XX-XX)

---

## FAQ

### Le PC n'obtient pas d'adresse IP
- Vérifier le câble réseau
- Exécuter ipconfig /release puis ipconfig /renew
- Contacter l'administrateur si le problème persiste

### Le PC a une adresse 169.254.x.x
- Le serveur DHCP n'est pas accessible
- Vérifier la connexion réseau
- Contacter l'administrateur

### Je dois changer l'adresse IP du PC
- Contacter l'administrateur pour créer une réservation DHCP
- Ou passer en IP statique (déconseillé pour les clients)