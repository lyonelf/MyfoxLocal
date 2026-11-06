# Myfox Local
Gestion hors cloud des centrales Myfox Zen et Home Control 2


# Se passer des serveurs Somfy : guide pas à pas

Ce guide explique comment faire fonctionner une centrale **Myfox / Somfy (Evology Zen, et en principe Home Control 2 / Pro)**
sans le cloud de Somfy, avec `myfox_local_serverAutoAprenant`, depuis la sauvegarde des noms jusqu'au tableau de bord.

> **Échéance.** L'arrêt définitif des serveurs Myfox HC2 est annoncé pour le **31 mars 2027**. Les étapes 1 et 3
> (récupérer les noms, observer le vrai serveur) ne sont possibles **que tant que le cloud répond**. Faites-les en premier.

> **Sécurité.** La liaison centrale ↔ serveur est en clair (TCP 8888, sans chiffrement ni authentification). Quiconque
> contrôle le serveur ou le DNS peut commander l'alarme, y compris la désarmer. Gardez le script **sur votre réseau local**,
> n'ouvrez jamais ses ports (8888 et 8081) sur Internet, et ne partagez jamais le dossier `myfox_data/`.

Avancement des tests : le script a été développé et validé sur une **Evology Zen**. Home Control 2 et Home Control Pro n'ont
pas été testées (seul le titre affiché change aujourd'hui) ; vos retours sont bienvenus.

---

## Vue d'ensemble

| # | Étape | Quand | Cloud nécessaire ? |
|---|---|---|---|
| 1 | Sauvegarder les noms via l'API Myfox | en premier | oui |
| 2 | Installer et lancer le script | | non |
| 3 | (Recommandé) Mode relais : observer le vrai serveur | avant la coupure | oui |
| 4 | Rediriger `core.steadyfox.com` vers votre machine | | non |
| 5 | Régler l'identité de la centrale (SN1 / SN2) | | non |
| 6 | Rattacher les noms et scénarios dans le tableau de bord | | non (relais pour les scénarios) |
| 7 | Passer en mode autonome et faire tourner en permanence | | non |

Ce qui change pour vous : l'application et le site officiels ne verront plus la centrale (« hors ligne »). Tout se pilote
alors depuis le tableau de bord du script. Les scénarios, eux, **restent exécutés par la centrale** elle-même.

---

## Étape 1 – Sauvegarder les noms depuis l'API Myfox

La centrale ne connaît que des **numéros**. Les noms (« Volet Salon »…) n'existent que dans le cloud.

### 1.1 Créer une application sur l'API Myfox

Il vous faut un **Client ID** et un **Client Secret** :

* Si vous utilisez déjà le plugin Jeedom *AlarmeMyfox*, ils sont dans la configuration de son équipement : passez à 1.2.
* Sinon, d'après les guides communautaires, connectez-vous sur **api.myfox.me**, ouvrez l'onglet **« My applications »**
  et créez une **« Personal application »** : le site affiche le Client ID et le Secret.
  *(Ces guides datent de 2014 à 2018 : l'écran a pu changer, et le site peut ne plus l'accepter. Vérifiez.)*

Il vous faut en plus l'**e-mail** et le **mot de passe** de votre compte Myfox habituel.

### 1.2 Lancer la sauvegarde

Sur n'importe quel PC avec Python 3 (pas besoin d'être sur le réseau de la centrale) :

```
python3 myfox_local_serverAutoAprenant.py --cloud-dump
```

Le script demande les quatre informations au clavier (secret et mot de passe invisibles) et crée un dossier
`myfox_data/myfox_backup` contenant uniquement les noms et identifiants des scénarios et équipements. Aucun mot de passe,
secret ou jeton n'est écrit. Détails, limites (les capteurs de sécurité ne sont pas lisibles par l'API) et dépannage :
voir **LISEZMOI_noms_du_cloud.md**.

Contrôle : `python3 myfox_local_serverAutoAprenant.py --backup-check`.

---

## Étape 2 – Installer et lancer le script

* Il faut **Python 3** seul : le script n'utilise que la bibliothèque standard, rien à installer.
* Placez le fichier dans un dossier dédié sur une machine **allumée en permanence** de votre réseau local
  (Raspberry Pi, mini-PC, NAS, box Jeedom…). Il crée `myfox_data/` à côté de lui.
* Donnez à cette machine une **adresse IP fixe** (réservation DHCP dans la box) : la centrale la contactera par là.
* Vérifiez que le pare-feu de la machine laisse entrer le **TCP 8888** (centrale) et **8081** (tableau de bord) depuis le réseau local.

Lancement en mode autonome :

```
python3 myfox_local_serverAutoAprenant.py
```

Le tableau de bord est alors sur `http://ADRESSE_DE_LA_MACHINE:8081`. Le port 8081 est le port par défaut du script ;
`--web-port` permet d'en choisir un autre. Autotest facultatif : `--selftest` doit afficher « 10 table(s) sur 10 reproduite(s) ».

Aide complète et protocole : `python3 myfox_local_serverAutoAprenant.py --reference`.

---

## Étape 3 – (Recommandé) Observer le vrai serveur en mode relais

Le mode relais fait passer la centrale **par votre script**, qui transmet tout au vrai serveur Somfy et enregistre tout.
L'application officielle continue de marcher. Il sert à :

* associer les **scénarios** à leurs noms (étape 6), car c'est le vrai site qui envoie la table du scénario à la centrale.

**Il faut connaître l'adresse IP du vrai serveur, pas son nom**, sinon le relais se résout lui-même vers lui-même une fois
le DNS redirigé. Avant de modifier le DNS, relevez-la :

```
nslookup core.steadyfox.com
```

Puis lancez :

```
python3 myfox_local_serverAutoAprenant.py --relay --upstream ADRESSE_IP_DU_VRAI_SERVEUR:8888
```

Si vous avez déjà redirigé le DNS (étape 4) et ne connaissez plus l'adresse réelle, interrogez un DNS public
(`nslookup core.steadyfox.com 1.1.1.1`) — tant que le nom existe encore.

---

## Étape 4 – Rediriger `core.steadyfox.com` vers votre machine

Au démarrage, la centrale résout **`core.steadyfox.com`** et ouvre une connexion TCP 8888. Le but : que ce nom pointe vers
la machine qui exécute le script. Cela se fait **dans votre réseau, sur votre DNS local** ; rien n'est modifié chez Somfy.

### 4.1 Créer l'enregistrement DNS local

Choisissez l'outil qui sert de DNS à la centrale (en général votre box ou un Pi-hole) :

* **Pi-hole** : *Local DNS → DNS Records* : domaine `core.steadyfox.com`, IP = celle de la machine du script.
* **dnsmasq** (OpenWrt, Raspberry Pi…) : ajouter la ligne `address=/core.steadyfox.com/IP_DE_LA_MACHINE`, puis redémarrer dnsmasq.
* **Routeur / box** : cherchez « DNS local », « hôtes statiques » ou « redirection DNS ». Beaucoup de box d'opérateur n'offrent
  pas cette fonction : dans ce cas, mettez un Pi-hole ou un dnsmasq et indiquez-le comme serveur DNS dans la box (DHCP), ou
  uniquement pour la centrale via une réservation DHCP avec DNS dédié si votre box le permet.

Vérification, depuis un PC qui utilise ce DNS :

```
nslookup core.steadyfox.com
```

La réponse doit être l'adresse de votre machine.

### 4.2 `hc2.steadyfox.com` (facultatif)

La centrale fait aussi, au démarrage, un petit contrôle HTTP (port 80) vers `hc2.steadyfox.com`. Elle n'en dépend pas pour
ouvrir la session sur le port 8888 ; sa fonction exacte est inconnue. Vous pouvez l'ignorer.

### 4.3 Redémarrer la centrale

La centrale ne relit le DNS qu'à l'ouverture d'une **nouvelle** session : **coupez l'alimentation secteur 10 secondes**
(sans toucher à la batterie), puis rétablissez-la.

### 4.4 Vérifier

* Dans la console du script, une ligne `OPEN` apparaît quand la centrale se connecte ; le tableau de bord affiche
  **« En ligne »** dans la carte *État*.
* Le tableau de bord indique le mode : **relais** (vous avez lancé `--relay`) ou **autonome**.
* La centrale se reconnecte seule en quelques secondes si le script redémarre.

---

## Étape 5 – Régler l'identité de la centrale (SN1 / SN2)

Chaque ordre envoyé par le script est signé d'un **identifiant d'émetteur = SN1 + SN2** (20 caractères), les deux numéros de
l'**étiquette sous la centrale**.

1. Ouvrez le tableau de bord et **cliquez sur le logo de la centrale** (carte *État*) : une fenêtre s'ouvre.
2. **Modèle** : Evology Zen, Home Control 2, Home Control Pro ou Autre. Cela ne change que le titre affiché.
3. **SN2** (12 caractères, c'est aussi l'adresse MAC) : lu automatiquement à la connexion de la centrale.
4. **SN1** (8 chiffres) : **à recopier à la main depuis l'étiquette**, une seule fois. Ni la centrale, ni le vrai serveur ne le
   transmettent : le vrai serveur signe ses ordres avec des valeurs `ffff9999…` qui ne le contiennent pas, et la centrale ne
   fait que renvoyer l'identifiant qu'on lui a envoyé. Si un relais voit un jour passer un SN1 réel, il est prérempli avec un avertissement.
5. **OK** enregistre : pas de redémarrage, la valeur s'applique à l'ordre suivant.

Tant que SN1 n'est pas saisi, le script signe avec `00000000`. On ne sait pas si la centrale contrôle cette valeur.
Option en ligne de commande : `--source-id` (20 caractères hexadécimaux) force l'identifiant.

---

## Étape 6 – Rattacher les noms dans le tableau de bord

Les identifiants du cloud n'ont **aucun rapport** avec les numéros de la centrale : c'est vous qui confirmez chaque lien.
Dans le tableau de bord, le bouton **☁ Noms dans le cloud** (pages *Mon système* et *Mes scénarios*) ouvre la fenêtre de rattachement.

* **Scénarios** (mode **relais** requis) : bouton *Assistant : associer les scénarios restants*. Pour chacun, ouvrez-le sur le
  site Myfox et cliquez **Enregistrer** sans rien modifier ; le site renvoie le scénario à la centrale, le script voit passer son
  numéro et l'associe. Si deux scénarios arrivent en même temps, l'assistant ne conclut pas : recommencez un par un.
* **Équipements** (mode **autonome** requis) : **Tester** fait bouger l'équipement ; repérez-le, choisissez son nom dans la liste
  et cliquez **Associer**. Une suggestion ★ aide quand l'ordre d'appairage est le même.
* Les **capteurs de sécurité** (détecteurs, tags, télécommandes, sirènes) ne sont pas lisibles par l'API : nommez-les à la main.

### Ajouter ou réintégrer une caméra (Panasonic BL-C121 relevée)

Sur le vrai site, ajouter une caméra envoie à la centrale l'ordre `621` ; la centrale se connecte alors **elle-même** à la caméra sur
le réseau local et rapporte sa progression au serveur. Le script reproduit cette séquence, ce qui évite d'être bloqué après la
fermeture du cloud.

1. Passez en **mode autonome** (la centrale doit être connectée) et vérifiez que la caméra est allumée et joignable sur le réseau local.
2. Sur la page *Tableau de bord*, carte *Mes caméras*, cliquez sur **➕ Ajouter une caméra**.
3. Saisissez l'**adresse MAC** (étiquette sous la caméra), l'**identifiant** et le **mot de passe** de la caméra, un nom et le numéro à lui donner.
   Cochez *Remplacer* si une caméra porte déjà ce numéro (le script envoie d'abord sa suppression).
4. Cliquez sur **Installer** et suivez l'avancement dans la fenêtre (environ 30 s, étapes 0 à 9).

Points à savoir :

* **C'est testé sur le papier, pas encore sur une centrale sans cloud** : la séquence et les réponses reproduisent un relevé complet fait
  en mode relais, mais le jeton de la caméra (8 caractères) que le vrai serveur génère est remplacé par une valeur aléatoire ; on ne sait
  pas si la centrale la vérifie. **Essayez tant que le cloud répond** (si cela échoue, repassez en relais et refaites l'ajout depuis le site).
* Le **mot de passe de la caméra circule en clair** sur votre réseau local (c'est le protocole de la centrale). Le script le masque dans ses journaux.
  Changez l'identifiant et le mot de passe par défaut de la caméra.
* L'option *serveur FTP d'alarme* (`tag-ftp-01.ig-1.net`) est une adresse du cloud Somfy ; elle ne servira plus après sa fermeture.
  Vos photos passent par la centrale (ordre 619) et n'en dépendent pas.

Ce que vous pouvez ensuite faire depuis le tableau de bord : armer/désarmer, piloter volets et prises, lancer et éditer les
scénarios, configurer les détecteurs, la caméra et la télécommande, consulter l'historique.

---

## Étape 7 – Passer en autonome et faire tourner en permanence

1. Arrêtez le relais et lancez le script **sans** `--relay` (ou changez le mode dans la carte *Mode* du tableau de bord).
2. Démarrage automatique :

   **Linux (systemd)** — exemple, adaptez les chemins et l'utilisateur :

   ```
   [Unit]
   Description=Serveur local Myfox
   After=network-online.target

   [Service]
   WorkingDirectory=/opt/myfox
   ExecStart=/usr/bin/python3 /opt/myfox/myfox_local_serverAutoAprenant.py
   Restart=always
   User=myfox

   [Install]
   WantedBy=multi-user.target
   ```

   puis `sudo systemctl enable --now myfox`.

   **Windows** : planificateur de tâches (« au démarrage », exécuter `python.exe` avec le chemin du script, redémarrer en cas
   d'échec), ou un gestionnaire de service tel que NSSM.
3. **Sauvegardez** régulièrement le dossier `myfox_data/` (noms, liens, apprentissage) : il contient aussi des horaires de
   présence, ne le partagez pas.

---

## Étape 8 – Sécuriser le tableau de bord (onglet « Sécurité »)

Le tableau de bord peut désarmer l'alarme et montrer la caméra : protégez-le dès que le script tourne en permanence.
Tout se règle dans l'onglet **Sécurité** (à droite de *Aide*).

1. **Créez le premier compte** (carte *Mon compte* : nom + mot de passe de 8 caractères minimum) : il est **super admin**. Tant qu'aucun compte
   n'existe, un bandeau d'avertissement reste affiché en haut de la page. Ensuite, chaque personne doit se connecter (12 heures, ou 30 jours
   avec « Rester connecté »). Après 5 échecs, l'adresse est bloquée 5 minutes. Mot de passe oublié : arrêtez le script et lancez
   `python3 myfox_local_serverAutoAprenant.py --reset-password` (supprime tous les comptes).
2. **Comptes et profils** : le super admin crée un compte par personne (cartes *Comptes* et *Profils et droits*) et lui donne un **profil**.
   Trois profils sont fournis : *Admin* (ajouter des appareils et des scénarios, photos, vidéos...), *Utilisateur* (armer, désarmer, lancer
   des scénarios) et *Invité* (consultation seule). La **matrice à cocher** permet de régler chaque droit, de renommer, supprimer ou ajouter
   des profils. *Super admin* a tous les droits, n'est pas modifiable et il en reste toujours au moins un. Les droits sont contrôlés par le
   serveur ; les boutons interdits sont simplement masqués.
3. **Désarmement** : le bouton *Désarmé* demande toujours une confirmation. L'option « Redemander le mot de passe du compte pour désarmer »
   y ajoute le mot de passe.
4. **Verrouillez le port de la centrale (8888)** : dans la carte *Centrale*, l'adresse vue sur le port 8888 est affichée.
   Réservez d'abord à la centrale une **adresse fixe** dans votre box ou routeur (« bail DHCP statique » ou « réservation d'adresse »),
   cochez la confirmation, puis cliquez sur *Verrouiller*. Plus aucune autre adresse ne pourra se faire passer pour la centrale.
   Si vous changez un jour son adresse, retirez le verrou avant.
5. **Journal d'audit** : en bas de l'onglet (fichier `audit.log` dans le dossier de données) : connexions, échecs, ordres envoyés,
   désarmements, refus.
6. **Jeedom / Home Assistant** : une fois un compte créé, ils doivent présenter une **clé d'accès** (onglet **Intégration**) qui porte un
   profil, et tout désarmement doit contenir `"confirm": true`. L'onglet donne les exemples complets.

Ce qui est refusé automatiquement : les adresses qui ne sont pas locales (tableau de bord et port 8888), les actions envoyées par un autre
site web, les noms d'hôte inattendus (option `--allow-host NOM` si vous accédez par un nom de domaine à vous).

> **Limites** : le tableau de bord est en HTTP non chiffré, donc le mot de passe circule en clair sur votre réseau local. Ne l'ouvrez
> **jamais** sur Internet par une redirection de port ; pour l'utiliser hors de chez vous, passez par un VPN (WireGuard, Tailscale...).
> Le port 8888 reste en clair : c'est la centrale qui l'impose.

---

## Retour arrière

Supprimez l'enregistrement DNS local de `core.steadyfox.com` et coupez l'alimentation de la centrale 10 secondes : elle
rejoindra le vrai serveur tant qu'il existe. Après le 31 mars 2027, il n'y aura plus de retour possible.

## Dépannage

| Symptôme | Piste |
|---|---|
| La centrale reste « Hors ligne » | `nslookup core.steadyfox.com` donne-t-il votre machine ? Centrale redémarrée après le changement ? Port 8888 ouvert dans le pare-feu ? |
| Elle apparaît puis disparaît en mode relais | Vous avez donné un **nom** à `--upstream` au lieu d'une adresse IP : le relais se contacte lui-même. |
| Les ordres sont ignorés | Vérifiez l'identité (étape 5), puis les messages `NOUVEAU` et les erreurs dans la console. |
| Le tableau de bord est injoignable | Port 8081 ouvert ? `--web-port` ou `--no-web` utilisé ? Adresse IP de la machine correcte ? |
| L'assistant de scénarios n'avance pas | Il faut le **mode relais** et un clic sur *Enregistrer* sur le site Myfox (10 min par scénario). |
| Le test d'un équipement n'agit pas | Il faut le **mode autonome** et la centrale connectée. |

## Ce qui n'est pas couvert

* L'application mobile officielle et les notifications Somfy ne fonctionneront plus : le script ne les remplace pas.
* Les comptes utilisateurs, pièces et historique du cloud ne sont pas migrés (seuls les noms le sont).
* La mise à jour du firmware de la centrale passait par le cloud : elle n'est plus possible.
* Caméra : l'ordre de délai reste à 0|0|0, probablement géré côté cloud.
