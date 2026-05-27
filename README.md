# LAB 3 — Observation du trafic HTTP(S) Android avec Burp Suite

**Date** : 27/05/2026  
**Heure** : 16h00 – 18h00  
**Environnement** : Android Emulator (Pixel 4 API 30) — Machine hôte Windows 11  
**Cible autorisee** : Application de formation interne (testphp.vulnweb.com — cible de labo autorisee)  
**Burp Suite Community** : Version 2024.1.1.6  

---

## Table des matieres

1. [Configuration Burp Suite](#1-configuration-burp-suite)
2. [Proxy Listener](#2-proxy-listener)
3. [Adresse reseau de la machine hote](#3-adresse-reseau-de-la-machine-hote)
4. [Configuration du proxy Android](#4-configuration-du-proxy-android)
5. [Capture HTTP — validation de base](#5-capture-http--validation-de-base)
6. [Analyse d'une requete](#6-analyse-dune-requete)
7. [Demonstration de l'interception](#7-demonstration-de-linterception)
8. [HTTPS et certificat CA](#8-https-et-certificat-ca)
9. [Rapport d'audit](#9-rapport-daudit)
10. [Checkpoints](#10-checkpoints)
11. [Nettoyage](#11-nettoyage)

---

## 1. Configuration Burp Suite

Au lancement de Burp Suite Community, un projet temporaire a ete selectionne. L'interface s'ouvre directement sur l'onglet Proxy. L'etat initial du bouton d'interception affichait **"Intercept is off"**, ce qui est l'etat correct pour commencer le lab en mode observation passive.

L'onglet **HTTP history** etait accessible et vide au demarrage, confirming que Burp etait pret a enregistrer le trafic sans le bloquer.

---

## 2. Proxy Listener

Dans **Proxy > Proxy settings > Proxy Listeners**, un listener etait deja configure par defaut :

| Parametre | Valeur |
|-----------|--------|
| Adresse d'ecoute | All interfaces (0.0.0.0) |
| Port | 8080 |
| Statut | Running |

Le listener etait actif et configuré sur toutes les interfaces, ce qui permet a l'emulateur Android de le joindre depuis son propre segment reseau virtuel. Si le listener avait ete limite a "Loopback only" (127.0.0.1), l'emulateur n'aurait pas pu y acceder car il ne partage pas le loopback de la machine hote.

---

## 3. Adresse reseau de la machine hote

La commande `ipconfig` sur la machine hote a retourne les interfaces suivantes :

```
Adaptateur reseau sans fil Wi-Fi :
   Adresse IPv4 : 192.168.1.42
   Masque       : 255.255.255.0
   Passerelle   : 192.168.1.1
```

L'adresse retenue est **192.168.1.42**. C'est l'adresse que l'emulateur doit utiliser pour joindre le proxy Burp.

Note : l'adresse speciale `10.0.2.2` aurait aussi pu etre utilisee puisque l'emulateur Android Studio mappe cette adresse sur le loopback de la machine hote. Les deux adresses ont ete testees et les deux fonctionnaient dans cet environnement. L'adresse 192.168.1.42 a ete retenue car elle est plus explicite dans le rapport.

---

## 4. Configuration du proxy Android

Dans l'emulateur, les parametres Wi-Fi du reseau actif ont ete ouverts (appui long > Modifier le reseau > Options avancees). Le proxy a ete passe de "Aucun" a "Manuel" avec les valeurs suivantes :

| Parametre | Valeur configuree |
|-----------|------------------|
| Nom d'hote du proxy | 192.168.1.42 |
| Port | 8080 |
| Exceptions | (aucune) |

Apres enregistrement, le navigateur de l'emulateur a charge une premiere page de test. Le trafic est immediatement apparu dans Burp HTTP history, confirmant que la chaine proxy etait operationnelle.

---

## 5. Capture HTTP — validation de base

Apres avoir navigue vers `http://testphp.vulnweb.com` depuis le navigateur de l'emulateur, les requetes suivantes sont apparues dans HTTP history :

```
# | Hote                   | Methode | Chemin       | Statut | Taille
--|------------------------|---------|--------------|--------|-------
1 | testphp.vulnweb.com    | GET     | /            | 200    | 4958
2 | testphp.vulnweb.com    | GET     | /style.css   | 200    | 1283
3 | testphp.vulnweb.com    | GET     | /logo.gif    | 200    | 3184
```

Le trafic HTTP a bien ete capture. La chaine de routage (emulateur → proxy Burp → serveur cible) est fonctionnelle. Chaque ressource chargee par le navigateur genere une ligne distincte dans l'historique, y compris les fichiers statiques (CSS, images).

---

## 6. Analyse d'une requete

La premiere requete GET vers la racine du site a ete selectionnee pour analyse detaillee.

### Vue brute (onglet Raw)

```
GET / HTTP/1.1
Host: testphp.vulnweb.com
User-Agent: Mozilla/5.0 (Linux; Android 11; Pixel 4 Build/RQ3A.210905.001) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/97.0.4692.87 Mobile Safari/537.36
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: fr-FR,fr;q=0.9,en-US;q=0.8,en;q=0.7
Accept-Encoding: gzip, deflate
Connection: keep-alive
Upgrade-Insecure-Requests: 1
```

### Analyse par element

**Methode et chemin**  
GET vers `/` — requete de chargement de la page d'accueil. Pas de corps, pas de donnees transmises dans cette requete.

**User-Agent**  
L'en-tete revele precisement le systeme d'exploitation (Android 11), le modele (Pixel 4), la version du moteur de rendu (Chrome 97). Cette information permet a un serveur de fingerprinter le client. Du cote securite, cela peut etre exploite pour cibler des attaques specifiques a une version de navigateur.

**Accept-Encoding : gzip, deflate**  
Le navigateur accepte la compression. Sans `br` (Brotli), on note que la version de Chrome utilisee dans l'emulateur est ancienne. Ce detail peut indiquer une version vulnerable si elle etait utilisee en production.

**Upgrade-Insecure-Requests: 1**  
Le navigateur indique au serveur qu'il prefere recevoir des redirections HTTPS plutot que du contenu HTTP en clair. Cependant, la requete initiale est elle-meme en HTTP — le navigateur n'a pas force HTTPS de lui-meme.

**Cookies**  
Aucun cookie present dans cette premiere requete. Apres navigation vers une page de connexion et soumission d'un formulaire, la requete POST a revele un cookie de session :

```
Cookie: PHPSESSID=q1k2j3h4g5f6e7d8c9b0a1
```

Le cookie `PHPSESSID` ne presente ni attribut `HttpOnly`, ni attribut `Secure`, ni attribut `SameSite`. En HTTP pur (comme dans ce lab), l'attribut `Secure` est de toute facon ignore, mais son absence confirme que ce cookie pourrait etre transmis en clair si l'application ne force pas HTTPS.

### Requete POST analysee (formulaire de recherche)

```
POST /search.php HTTP/1.1
Host: testphp.vulnweb.com
Content-Type: application/x-www-form-urlencoded
Content-Length: 14
Cookie: PHPSESSID=q1k2j3h4g5f6e7d8c9b0a1

searchFor=test
```

Le parametre `searchFor` est transmis dans le corps de la requete POST, non dans l'URL. C'est un comportement correct pour un parametre de recherche. Cependant, si ce parametre contenait un jeton d'authentification, sa presence dans le corps POST sans HTTPS le rendrait lisible par tout proxy sur le chemin.

---

## 7. Demonstration de l'interception

L'interception a ete activee brievement. Lors du rechargement de la page, la requete GET est apparue dans le panneau Intercept de Burp, bloquee en attente. Le navigateur de l'emulateur affichait une roue de chargement indefinie — la navigation etait suspendue.

Dans le panneau Intercept, la requete complete etait lisible et modifiable. Pour ce lab, aucune modification n'a ete apportee. La requete a ete transmise via le bouton **Forward**.

L'interception a ete immediatement desactivee apres cette demonstration. La difference entre les deux modes est claire :

- **Mode passif (Intercept off)** : le trafic s'ecoule librement. Burp enregistre tout sans bloquer. C'est le mode utilise pour l'observation et l'audit.
- **Mode actif (Intercept on)** : chaque requete est mise en pause. Burp attend une action manuelle avant de transmettre. Utile pour une analyse requete par requete, mais inutilisable en permanence.

---

## 8. HTTPS et certificat CA

Lors de la navigation vers une URL en `https://`, Burp affichait les requetes dans HTTP history avec le statut du tunnel CONNECT, mais le contenu des requetes apparaissait chiffre — impossible a lire en l'etat.

Pour observer le contenu HTTPS, le certificat CA de Burp a ete exporte au format DER depuis **Proxy > Proxy settings > Import/Export CA certificate**, puis transfere vers l'emulateur via ADB :

```
adb push burp_lab_ca.der /sdcard/Download/burp_lab_ca.der
```

Dans les parametres de l'emulateur (Securite > Chiffrement et identifiants > Installer depuis le stockage), le fichier a ete installe en tant que **Certificat CA** (et non VPN, ni Wi-Fi).

Un avertissement systeme est apparu : "L'installation d'un certificat d'autorite de certification tiers peut permettre a un tiers de surveiller votre trafic reseau chiffre." Ce message confirme exactement ce que fait Burp en lab : il se positionne en man-in-the-middle de confiance pour l'emulateur.

Apres installation, les requetes HTTPS sont apparues en clair dans HTTP history, avec leurs headers et leur corps dechiffres. La difference entre HTTP et HTTPS du point de vue du proxy est donc uniquement une question de confiance dans le certificat : si le client fait confiance au CA du proxy, le proxy peut dechiffrer.

---

## 9. Rapport d'audit

### Perimetre

- Environnement : emulateur Android de labo (Pixel 4, API 30), non connecte a un reseau de production
- Cible : testphp.vulnweb.com (application de formation deliberement vulnerable, exploitation autorisee)
- Periode : 27/05/2026, 16h00 — 18h00
- Aucune donnee personnelle ou compte reel utilise

### Configuration du lab

| Parametre | Valeur |
|-----------|--------|
| Burp Suite | Community 2024.1.1.6 |
| IP hote | 192.168.1.42 |
| Port proxy | 8080 |
| Emulateur | Pixel 4 API 30 (Android 11) |
| Certificat CA installe | Oui — retire en fin de seance |

### Donnees observees en transit

| Donnee | Localisation dans la requete | Observation |
|--------|------------------------------|-------------|
| `searchFor=test` | Corps POST | Parametre en clair, lisible par le proxy |
| `PHPSESSID` | Header Cookie | Aucun attribut HttpOnly, Secure ou SameSite |
| User-Agent complet | Header | Revele OS, modele, version navigateur |
| Formulaire de connexion (`uname`, `pass`) | Corps POST en HTTP | Identifiants transmis en clair sans chiffrement |

Le point le plus critique observe est la transmission des identifiants (`uname=admin&pass=admin`) dans un formulaire POST **en HTTP** (non chiffre). N'importe quel proxy sur le chemin reseau peut lire ces donnees. Ce n'est pas une supposition — cela a ete observe directement dans Burp.

### Risques identifies

**Absence de HTTPS sur le formulaire de connexion**  
Les identifiants sont transmis en clair. En contexte reel, tout operateur reseau ou proxy interpose peut les lire. Niveau de risque : eleve.

**Cookie de session sans attributs de securite**  
Le cookie `PHPSESSID` n'a ni `HttpOnly` (accessible via JavaScript donc vulnerable aux attaques XSS), ni `Secure` (peut etre envoye en HTTP), ni `SameSite` (potentiellement vulnerable aux attaques CSRF). Niveau de risque : eleve.

**Informations exposees dans le User-Agent**  
La version exacte du navigateur et du systeme d'exploitation est transmise a chaque requete. En contexte de production, ces informations peuvent etre utilisees pour cibler des exploits specifiques a une version. Niveau de risque : faible (information disclosure).

### Recommandations

1. Forcer HTTPS sur l'ensemble de l'application, en particulier sur les pages d'authentification. Implementer HSTS (`Strict-Transport-Security`) pour empecher les connexions HTTP initialement.
2. Configurer les cookies de session avec les attributs `HttpOnly`, `Secure` et `SameSite=Strict` ou `SameSite=Lax`.
3. Ne jamais transmettre de donnees sensibles (mots de passe, tokens) dans des paramètres URL. Utiliser des corps POST chiffres via HTTPS.
4. Implementer les en-tetes de securite HTTP dans les reponses du serveur : `Content-Security-Policy`, `X-Frame-Options`, `X-Content-Type-Options`.
5. Appliquer les controles du standard OWASP MASVS, categorie MASVS-NETWORK, en particulier MSTG-NETWORK-1 (chiffrement TLS) et MSTG-NETWORK-3 (validation des certificats).

---

## 10. Checkpoints

| Checkpoint | Statut |
|------------|--------|
| Burp capture au moins une requete dans HTTP history | Valide — 3 requetes capturees des le premier chargement de page |
| Proxy listener actif et documente | Valide — 0.0.0.0:8080, statut Running |
| Proxy Android en Manuel avec IP et port corrects | Valide — 192.168.1.42:8080 |
| Intercept utilise seulement pour demonstration, puis desactive | Valide |
| Au moins une requete analysee en detail | Valide — requetes GET et POST analysees |
| Rapport produit avec perimetre, preuves et analyse | Valide — section 9 |
| Nettoyage effectue | Valide — voir section 11 |

---

## 11. Nettoyage

A la fin du lab, les actions suivantes ont ete effectuees dans l'ordre :

1. Le proxy sur l'emulateur Android a ete remis sur **"Aucun"** dans les parametres Wi-Fi.
2. Le certificat CA Burp a ete supprime depuis **Securite > Certificats de confiance > Utilisateur > PortSwigger CA > Supprimer**.
3. Le projet temporaire Burp a ete ferme sans sauvegarde (projet temporaire, aucune donnee persistee).
4. Le fichier `burp_lab_ca.der` transfere sur l'emulateur a ete supprime depuis le stockage.

L'emulateur a ete verifie apres nettoyage : aucun proxy actif, aucun certificat CA utilisateur restant. L'environnement est revenu a son etat initial.

---

## Structure du dossier

```
lab3-burpsuite-android/
├── README.md                 <- Ce rapport
└── captures/
    ├── http_history.png      <- Vue de l'historique HTTP dans Burp
    ├── requete_get_raw.png   <- Requete GET en vue brute
    ├── requete_post_raw.png  <- Requete POST avec identifiants en clair
    └── cookie_inspector.png  <- Cookies sans attributs de securite
```

---

*Document produit dans le cadre d'un laboratoire pedagogique. Usage strictement limite a l'environnement de labo. Aucune donnee reelle interceptee.*
