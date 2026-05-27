# 🔬 LAB 3 — Observation du trafic HTTP(S) Android avec Burp Suite

> **Contexte pédagogique** · Sécurité Mobile · Niveau Débutant–Intermédiaire  
> **Durée estimée** : 2 h 00  
> **Date** : *(à renseigner)*  
> **Formateur / Module** : *(à renseigner)*

---

## 📋 Sommaire

1. [Vue d'ensemble](#-vue-densemble)
2. [Objectifs pédagogiques](#-objectifs-pédagogiques)
3. [Prérequis](#-prérequis)
4. [Règles de sécurité](#️-règles-de-sécurité--obligatoires)
5. [Architecture du lab](#️-architecture-du-lab)
6. [Étape 1 — Préparer Burp Suite](#-étape-1--préparer-burp-suite-projet-et-mode-proxy)
7. [Étape 2 — Vérifier le Proxy Listener](#-étape-2--vérifier-le-proxy-listener-adresse-et-port)
8. [Étape 3 — Identifier l'adresse réseau hôte](#-étape-3--identifier-ladresse-réseau-de-la-machine-hôte)
9. [Étape 4 — Configurer le proxy Android](#-étape-4--configurer-le-proxy-côté-android-emulator)
10. [Étape 5 — Premier test HTTP](#-étape-5--premier-test--capturer-du-http-validation-de-base)
11. [Étape 6 — Lire une requête comme un analyste](#-étape-6--lire-une-requête-comme-un-analyste-sans-modification)
12. [Étape 7 — Interception contrôlée](#-étape-7--interception-contrôlée-mode-pédagogique)
13. [Étape 8 — HTTPS et certificat CA](#-étape-8--https-en-laboratoire--principe-du-certificat-ca)
14. [Étape 9 — Mini-rapport d'audit](#-étape-9--produire-un-mini-rapport-preuve--contexte)
15. [Checkpoints de validation](#-checkpoints-de-validation)
16. [Nettoyage de fin de séance](#-fin-de-lab--nettoyage-hygiène)
17. [Glossaire](#-glossaire)
18. [Ressources complémentaires](#-ressources-complémentaires)

---

## 🔭 Vue d'ensemble

Ce laboratoire met en place un **proxy d'observation** entre un Android Emulator et une application cible autorisée (site de test ou application de formation). L'outil central est **Burp Suite Community**, un proxy HTTP(S) d'analyse largement utilisé en tests de sécurité web et mobile.

### Pourquoi ce lab ?

Les applications mobiles communiquent avec des serveurs via le protocole HTTP ou HTTPS. Ces échanges contiennent des informations critiques : jetons d'authentification, paramètres de requête, cookies de session. Comprendre comment observer ce trafic — de manière passive, sans modification — est une compétence fondamentale en sécurité mobile.

```
┌─────────────────────────────────────────────────────────┐
│  FLUX NORMAL                                            │
│  [Android App] ──────────────────────▶ [Serveur Cible]  │
│                                                         │
│  FLUX EN LABORATOIRE (proxy interposé)                  │
│  [Android App] ──▶ [Burp Suite Proxy] ──▶ [Serveur]     │
│                           │                             │
│                    Observation / Log                    │
└─────────────────────────────────────────────────────────┘
```

---

## 🎯 Objectifs pédagogiques

À la fin de ce lab, vous serez capable de :

| # | Compétence | Indicateur de réussite |
|---|------------|------------------------|
| 1 | **Configurer le proxy** | Burp capture au moins une requête dans HTTP history |
| 2 | **Analyser une requête** | Identifier méthode, URL, headers, cookies, paramètres |
| 3 | **Expliquer HTTP vs HTTPS** | Décrire le rôle du certificat CA en contexte labo |
| 4 | **Utiliser l'interception** | Activer/désactiver sans bloquer le trafic durablement |
| 5 | **Produire une trace d'audit** | Rédiger une fiche de preuve contextuelle reproductible |

---

## 🛠️ Prérequis

### Logiciels requis

| Outil | Version recommandée | Rôle |
|-------|--------------------|----|
| **Burp Suite Community** | ≥ 2023.x | Proxy HTTP(S) d'analyse |
| **Android Studio** | ≥ Hedgehog | Gestionnaire d'émulateur Android |
| **Android Emulator** | API 29–33 (Pixel/Nexus) | Périphérique Android virtuel |
| **Navigateur de l'émulateur** | Chrome / WebView intégré | Client HTTP de test |

### Connaissances préalables

- [ ] Notions de base sur le protocole HTTP (méthodes, codes de réponse, headers)
- [ ] Familiarité avec l'interface Android (paramètres Wi-Fi)
- [ ] Compréhension basique des notions de proxy réseau
- [ ] Lecture d'une adresse IP (format `A.B.C.D`)

### Environnement réseau

- Réseau de labo isolé **ou** réseau local simple
- Machine hôte et émulateur sur le **même segment réseau**
- Accès à une **cible autorisée** (site de test interne, maquette locale, appli de formation)

> ⚠️ **Important** : Aucun test ne doit être effectué sur des applications ou sites en production.

---

## 🛡️ Règles de sécurité — OBLIGATOIRES

```
╔══════════════════════════════════════════════════════════════╗
║               RÈGLES DU LABORATOIRE                          ║
╠══════════════════════════════════════════════════════════════╣
║  ✅ Trafic intercepté uniquement pour des cibles autorisées  ║
║  ✅ Aucun compte personnel, aucune donnée sensible           ║
║  ✅ Certificat de labo retiré à la fin de la séance          ║
║  ✅ Proxy désactivé sur l'émulateur après le lab             ║
║  ❌ Ne jamais intercepter du trafic en production            ║
║  ❌ Ne jamais installer un certificat CA sur un vrai device  ║
╚══════════════════════════════════════════════════════════════╝
```

---

## 🏗️ Architecture du lab

```
Machine Hôte (Windows / Linux / macOS)
┌──────────────────────────────────────────────────┐
│                                                  │
│  ┌─────────────────┐    ┌─────────────────────┐  │
│  │  Burp Suite     │    │  Android Studio      │  │
│  │  Community      │    │  (AVD Manager)       │  │
│  │                 │    │                      │  │
│  │  Listener:      │    │  ┌───────────────┐   │  │
│  │  <IP_HOTE>:     │◀───│  │ Android       │   │  │
│  │  <PORT_PROXY>   │    │  │ Emulator      │   │  │
│  │                 │    │  │               │   │  │
│  │  HTTP History   │    │  │ Proxy Manuel: │   │  │
│  │  Intercept      │    │  │ <IP_HOTE>:    │   │  │
│  └────────┬────────┘    │  │ <PORT_PROXY>  │   │  │
│           │             │  └───────────────┘   │  │
│           │             └─────────────────────-┘  │
└───────────┼──────────────────────────────────────-┘
            │
            ▼
   [Serveur Cible Autorisé]
   (réseau de labo / maquette locale)
```

**Paramètres à documenter en début de séance :**

| Paramètre | Valeur |
|-----------|--------|
| `<IP_HOTE>` | *(à noter à l'étape 3)* |
| `<PORT_PROXY>` | *(à noter à l'étape 2)* |
| URL cible autorisée | *(à définir avec le formateur)* |
| Version Burp Suite | *(à noter)* |
| Date / Heure | *(à noter)* |

---

## 🔵 Étape 1 — Préparer Burp Suite (projet et mode Proxy)

### Objectif
Lancer Burp Suite et valider que l'interface est en mode observation passive.

### Procédure

1. **Lancer Burp Suite Community** depuis le menu applications.
2. À l'écran d'accueil, sélectionner **"Temporary project"** → cliquer **Next**.
3. Conserver la configuration par défaut → cliquer **Start Burp**.
4. Naviguer vers l'onglet **Proxy** dans la barre supérieure.
5. Vérifier que la page **"Intercept"** est visible.
6. S'assurer que le bouton affiche **"Intercept is off"** *(état initial voulu)*.
7. Cliquer sur **"HTTP history"** pour vérifier que la liste est accessible (vide pour l'instant).

### Pourquoi ?

> Un proxy d'observation ne doit **pas bloquer le trafic** tant que la configuration réseau n'est pas validée de bout en bout. Démarrer en mode passif évite de se retrouver bloqué par des requêtes en attente alors que la configuration n'est pas encore opérationnelle.

### ✅ À observer

- [ ] L'onglet **HTTP history** est présent et accessible
- [ ] Le bouton d'interception indique **"Intercept is off"**
- [ ] L'interface Burp est stable et réactive

### ❌ Erreurs fréquentes

| Erreur | Cause | Correction |
|--------|-------|------------|
| Trafic bloqué dès le départ | Intercept activé par défaut | Cliquer "Intercept is off" pour le désactiver |
| Confusion entre onglets | Première utilisation de Burp | Rester sur Proxy > HTTP history pour ce lab |

### 💡 Remarque

> Un environnement de labo doit rester stable et reproductible. Si l'outil bloque le trafic avant validation, l'expérience devient frustrante et les résultats ne sont pas fiables.

---

## 🔵 Étape 2 — Vérifier le Proxy Listener (adresse et port)

### Objectif
Identifier et documenter le port d'écoute du proxy Burp Suite.

### Procédure

1. Dans Burp Suite, aller dans **Proxy → Proxy settings** (ou **Options** selon la version).
2. Localiser la section **"Proxy Listeners"**.
3. Vérifier qu'un listener est listé avec le statut **"Running"** ✓.
4. **Noter** l'adresse d'écoute et le port :

```
Port du proxy    : <PORT_PROXY>   (ex. 8080)
Adresse d'écoute : ___________   ("Loopback only" ou "All interfaces")
```

5. Si nécessaire, configurer l'adresse en **"All interfaces"** pour que l'émulateur puisse joindre le proxy.

### Pourquoi ?

> Le téléphone/émulateur doit savoir où envoyer son trafic. Il a besoin d'une adresse IP (celle de la machine hôte) **et** d'un port. Le listener Burp doit être configuré pour accepter des connexions provenant d'un hôte non-local.

### ✅ À observer

- [ ] Un listener avec le statut **"Running"** (icône verte ou coche)
- [ ] Le port est défini et noté sous `<PORT_PROXY>`
- [ ] L'adresse d'écoute est compatible avec l'émulateur

### ❌ Erreurs fréquentes

| Erreur | Symptôme | Correction |
|--------|----------|------------|
| Listener désactivé | Statut "Off" ou absent | Cocher "Running" / "Enabled" |
| Listener limité au loopback | L'émulateur ne peut pas joindre le proxy | Changer en "All interfaces" ou spécifier l'IP hôte |
| Port déjà utilisé | Erreur au démarrage du listener | Changer le port (ex. 8081, 8888) |

### 💡 Remarque

> L'adresse et le port sont des **paramètres de labo** critiques. Les documenter avec soin permet de reproduire la séance ou de dépanner un problème de connexion rapidement.

---

## 🔵 Étape 3 — Identifier l'adresse réseau de la machine hôte

### Objectif
Obtenir l'adresse IPv4 de la machine hôte, accessible depuis l'émulateur Android.

### Procédure

**Sur Windows :**
```powershell
# Dans PowerShell ou l'invite de commandes
ipconfig

# Chercher l'interface réseau active (Wi-Fi ou Ethernet du lab)
# et noter l'adresse "Adresse IPv4"
```

**Sur Linux / macOS :**
```bash
ip addr show
# ou
ifconfig
```

**Exemple de sortie attendue :**
```
Adaptateur réseau sans fil Wi-Fi :
   Adresse IPv4. . . . . . . . . . . . . .: 192.168.X.Y
   Masque de sous-réseau . . . . . . . . .: 255.255.255.0
   Passerelle par défaut . . . . . . . . .: 192.168.X.1
```

**Résultat à noter :**
```
<IP_HOTE> = ___.___.___.___ 
```

> 💡 **Cas particulier Android Emulator** : Si l'émulateur est lancé depuis Android Studio sur la même machine, l'adresse spéciale `10.0.2.2` pointe vers le loopback de la machine hôte. Vérifier si cette adresse fonctionne dans votre configuration avant d'utiliser l'IP locale.

### Pourquoi ?

> Le proxy Burp s'exécute sur la machine hôte. L'émulateur Android doit pouvoir la joindre via une adresse réseau valide. Sans cette adresse, l'émulateur ne sait pas où diriger son trafic.

### ✅ À observer

- [ ] Une adresse IPv4 du format `A.B.C.D` est identifiée
- [ ] Cette adresse appartient au même réseau que l'émulateur
- [ ] Aucune interface VPN inactive ou IP publique n'est confondue

### ❌ Erreurs fréquentes

| Erreur | Conséquence | Correction |
|--------|-------------|------------|
| Utiliser l'IP d'une interface VPN inactive | Connexion impossible | Identifier la bonne interface réseau de labo |
| Confondre IP publique et IP locale | Proxy injoignable depuis l'émulateur | Utiliser toujours une IP du réseau local |
| Plusieurs interfaces actives | Ambiguïté sur l'adresse à utiliser | Désactiver les interfaces non utilisées si possible |

### 💡 Remarque

> En laboratoire, un réseau simple avec peu d'interfaces actives réduit considérablement les erreurs de configuration. Moins d'interfaces = moins d'ambiguïté.

---

## 🔵 Étape 4 — Configurer le proxy côté Android Emulator

### Objectif
Diriger le trafic réseau de l'émulateur Android vers Burp Suite.

### Procédure

1. Dans l'**Android Emulator**, ouvrir l'application **Paramètres** (Settings).
2. Aller dans **Réseau et Internet** → **Wi-Fi**.
3. Appuyer longuement sur le réseau Wi-Fi actif → **Modifier le réseau**.
   *(Ou cliquer sur l'icône ⚙️ à côté du réseau)*
4. Déplier les **Options avancées**.
5. Dans le champ **Proxy**, sélectionner **Manuel** (Manual).
6. Renseigner les champs :

```
Nom d'hôte du proxy  : <IP_HOTE>      (ex. 192.168.1.X ou 10.0.2.2)
Port du proxy        : <PORT_PROXY>   (ex. 8080)
```

7. Laisser le champ **"Ne pas utiliser de proxy pour"** vide.
8. Appuyer sur **Enregistrer**.

### Vérification rapide

Après configuration, ouvrir le navigateur de l'émulateur et accéder à `http://burpsuite` ou à votre cible autorisée. Si la configuration est correcte, une réponse ou une erreur "Connection refused" indique que le trafic transite bien par le proxy.

### Pourquoi ?

> Sans proxy configuré, le trafic du navigateur part directement vers Internet et **n'apparaît jamais dans Burp**. La configuration manuelle du proxy force l'émulateur à envoyer toutes ses requêtes HTTP(S) à Burp avant de les transmettre au serveur.

### ✅ À observer

- [ ] Le champ Proxy affiche bien **"Manuel"** après configuration
- [ ] Le nom d'hôte correspond à `<IP_HOTE>` noté à l'étape 3
- [ ] Le port correspond à `<PORT_PROXY>` noté à l'étape 2

### ❌ Erreurs fréquentes

| Erreur | Symptôme | Correction |
|--------|----------|------------|
| Mauvais port | Aucun trafic dans Burp | Revérifier `<PORT_PROXY>` |
| Hostname vide ou incorrect | Échec de connexion réseau | Ressaisir `<IP_HOTE>` exactement |
| Configuration sur le mauvais réseau Wi-Fi | Proxy ignoré | Vérifier que le réseau configuré est bien le réseau actif |
| Proxy en mode "Auto" | Fichier PAC ignoré ou proxy absent | Sélectionner "Manuel" explicitement |

### 💡 Remarque

> Le proxy configuré à ce niveau s'applique au trafic du **navigateur** et de nombreuses applications système. En labo, limiter les tests à la cible autorisée désignée. Ne pas "surfer" librement avec le proxy actif.

---

## 🔵 Étape 5 — Premier test : capturer du HTTP (validation de base)

### Objectif
Valider la chaîne complète de capture avec un trafic HTTP simple (non chiffré).

### Procédure

1. Dans l'émulateur Android, ouvrir le **navigateur**.
2. Naviguer vers la **cible autorisée HTTP** (ex. `http://testsite.local` ou l'URL fournie par le formateur).
3. Attendre que la page se charge.
4. Dans Burp Suite, ouvrir **Proxy → HTTP history**.
5. Vérifier l'apparition d'au moins une requête dans la liste.

### Ce que vous devriez voir dans HTTP history

```
# | Host           | Method | URL            | Params | Edited | Status | Length | MIME type | Extension | Time
--|----------------|--------|----------------|--------|--------|--------|--------|-----------|-----------|------
1 | testsite.local | GET    | /index.html    |        |        | 200    | 1024   | HTML      | html      | 14:23:01
```

### Pourquoi ?

> Cette étape confirme que le **routage via proxy fonctionne** de bout en bout. En validant d'abord avec HTTP (sans chiffrement), on s'assure que la configuration réseau est correcte avant d'aborder la complexité d'HTTPS.

### ✅ À observer

- [ ] Au moins **une ligne** apparaît dans HTTP history
- [ ] La méthode (**GET** ou **POST**) est visible
- [ ] L'URL correspond à la cible naviguée
- [ ] Le code de statut est visible (200, 301, 404, etc.)
- [ ] La taille de la réponse est indiquée

### ❌ Erreurs fréquentes

| Erreur | Diagnostic | Action |
|--------|------------|--------|
| HTTP history vide | Proxy non actif ou mal configuré | Revoir étapes 2, 3 et 4 |
| Requêtes bloquées (Intercept actif) | Le trafic reste "en attente" | Désactiver l'interception (étape 1) |
| Erreur réseau dans le navigateur | Burp non démarré ou listener down | Vérifier que Burp est lancé et le listener actif |

### 💡 Remarque

> Si rien n'apparaît après 1–2 minutes : **Stop → Diagnostic**. Reprendre méthodiquement depuis l'étape 2. Un lab qui fonctionne est un lab qu'on a validé étape par étape.

---

## 🔵 Étape 6 — Lire une requête comme un analyste (sans modification)

### Objectif
Comprendre la structure d'une requête HTTP capturée et identifier les éléments de sécurité pertinents.

### Procédure

1. Dans **HTTP history**, cliquer sur une requête capturée.
2. Observer le panneau du bas (split view) :

#### Onglet "Raw" — Vue brute
```
GET /search?q=test&lang=fr HTTP/1.1
Host: testsite.local
User-Agent: Mozilla/5.0 (Linux; Android 11; Pixel 4) ...
Accept: text/html,application/xhtml+xml,...
Accept-Language: fr-FR,fr;q=0.9
Cookie: session_id=abc123xyz; theme=dark
Connection: keep-alive
```

#### Onglet "Inspector" ou "Parsed" — Vue structurée

| Section | Contenu observé |
|---------|----------------|
| **Request line** | Méthode + chemin + version HTTP |
| **Query Parameters** | `q=test`, `lang=fr` |
| **Headers** | User-Agent, Accept, Cookie... |
| **Cookies** | `session_id`, `theme` |
| **Body** | (vide pour GET, données pour POST) |

### Éléments à analyser du point de vue sécurité

| Élément | Ce qu'on cherche | Risque potentiel |
|---------|-----------------|-----------------|
| **Paramètres en URL** | Données sensibles en clair | Token, mot de passe exposé dans les logs serveur |
| **Cookie `session_id`** | Attributs `HttpOnly`, `Secure`, `SameSite` | Absence = vulnérabilité XSS ou CSRF possible |
| **User-Agent** | Informations sur le client | Fingerprinting, compatibilité |
| **En-têtes de sécurité** | Présence/absence dans la **réponse** | `X-Frame-Options`, `CSP`, `HSTS` |

### Pourquoi ?

> La compétence clé en sécurité mobile est l'**analyse** : comprendre ce qui est envoyé, à quelle fréquence, et pourquoi. La modification des requêtes est hors périmètre de ce lab — on reste en mode lecture.

### ✅ À observer

- [ ] La méthode HTTP (GET / POST) est identifiée
- [ ] Le chemin et les paramètres sont lisibles
- [ ] Au moins un header est noté et analysé
- [ ] Les cookies sont listés (présence / absence d'attributs)

### ❌ Erreurs fréquentes

| Erreur | Impact |
|--------|--------|
| Observer uniquement le corps (body) | Manquer les headers critiques |
| Interpréter un cookie comme "secret" sans vérifier ses attributs | Conclusion erronée |
| Ne pas noter le contexte (version app, date, cible) | Trace d'audit non reproductible |

### 💡 Remarque

> Le trafic observé est une **preuve**. Une preuve sans contexte n'a aucune valeur dans un rapport. Toujours noter : version de l'application testée, environnement de labo, date et heure, URL cible.

---

## 🔵 Étape 7 — Interception contrôlée (mode pédagogique)

### Objectif
Comprendre le principe du mode interception (bloquant) par opposition au mode observation passif.

### Procédure

1. Dans Burp Suite → **Proxy → Intercept**.
2. Cliquer sur le bouton pour activer : **"Intercept is on"**.
3. Dans l'émulateur, **rafraîchir** la page de la cible autorisée (F5 ou bouton actualiser).
4. Observer dans Burp : la requête apparaît dans le panneau d'interception et est **"en attente"**.
5. Lire les informations affichées.
6. Cliquer sur **"Forward"** pour transmettre la requête (ou "Drop" pour l'abandonner).
7. Immédiatement après l'observation, **désactiver l'interception** : cliquer pour revenir à **"Intercept is off"**.

### Comparaison des modes

| Mode | Comportement | Usage |
|------|-------------|-------|
| **Passif** (Intercept off) | Le trafic s'écoule librement, Burp enregistre tout dans HTTP history | Observation, audit, inventaire |
| **Actif** (Intercept on) | Chaque requête est mise en attente, analyse requête par requête | Inspection ciblée, démonstration |

### Pourquoi ?

> Comprendre la **différence entre passif et actif** est fondamental. En labo débutant, la priorité est la **lecture et la preuve**, pas la manipulation. L'interception est présentée ici à titre de démonstration uniquement.

### ✅ À observer

- [ ] La requête apparaît dans le panneau Intercept (en attente)
- [ ] La navigation est bloquée dans le navigateur pendant ce temps
- [ ] Après "Forward", la navigation reprend normalement
- [ ] L'interception est **désactivée immédiatement** après la démonstration

### ❌ Erreurs fréquentes

| Erreur | Conséquence |
|--------|-------------|
| Laisser l'intercept activé | Toute la navigation est bloquée |
| Utiliser "Drop" au lieu de "Forward" | La requête est abandonnée, la page ne charge pas |
| Modifier la requête pendant l'interception | Hors périmètre de ce lab — à éviter |

### 💡 Remarque

> L'interception est un outil puissant. **En formation débutant, son usage est limité à la démonstration.** Retourner systématiquement en mode passif après chaque démonstration d'interception.

---

## 🔵 Étape 8 — HTTPS en laboratoire : principe du certificat CA

### Objectif
Comprendre pourquoi HTTPS nécessite une configuration supplémentaire et comment la gérer proprement en labo.

### Contexte théorique

```
TRAFIC HTTP  : données en clair → Burp peut lire directement
TRAFIC HTTPS : données chiffrées → Burp a besoin d'un certificat pour déchiffrer
```

Pour que le navigateur Android accepte le proxy en HTTPS sans afficher d'avertissement, le certificat CA de Burp doit être **reconnu comme autorité de confiance** dans l'émulateur.

### Procédure (si HTTPS est dans le périmètre du lab)

#### a) Exporter le certificat CA Burp
1. Dans Burp Suite → **Proxy → Proxy settings → Import / export CA certificate**.
2. Sélectionner **"Export certificate in DER format"**.
3. Sauvegarder sous `burp_lab_ca.der` dans un dossier accessible.

#### b) Transférer le certificat vers l'émulateur
```bash
# Depuis la machine hôte (Android Debug Bridge)
adb push burp_lab_ca.der /sdcard/Download/burp_lab_ca.der
```

#### c) Installer le certificat sur l'émulateur
1. Dans l'émulateur → **Paramètres → Sécurité → Chiffrement et identifiants**.
2. Choisir **"Installer depuis le stockage"** → sélectionner le fichier `.der`.
3. Sélectionner le type : **"Certificat CA"** *(et non VPN, Wi-Fi, ou appli)*.
4. Accepter l'avertissement système.

#### Types de certificats proposés — ne pas confondre

| Type | Usage | À sélectionner pour ce lab |
|------|-------|---------------------------|
| **Certificat CA** | Autorité racine de confiance | ✅ Oui |
| **VPN & App user certificate** | Certificat client personnel | ❌ Non |
| **Wi-Fi certificate** | Authentification réseau 802.1X | ❌ Non |

### Pourquoi ?

> HTTPS protège les données **contre l'interception**. C'est son rôle principal. Pour qu'un proxy puisse analyser ce trafic en labo, il faut un modèle de confiance contrôlé. Sans le certificat CA, le navigateur affiche une erreur SSL et refuse la connexion — comportement attendu et correct en production.

### ✅ À observer

- [ ] La distinction entre les types de certificats est claire
- [ ] L'avertissement système lors de l'ajout d'une autorité CA est noté
- [ ] Le trafic HTTPS apparaît dans HTTP history (si applicable)

### ❌ Erreurs fréquentes

| Erreur | Risque | Correction |
|--------|--------|------------|
| Installer le certificat CA sur un téléphone personnel | Réduction permanente de la sécurité | Strictement réservé à l'émulateur de labo |
| Oublier de retirer le certificat à la fin | Persistance d'une faille de sécurité | Voir section "Nettoyage" |
| Installer un mauvais type de certificat | HTTPS toujours non déchiffré | Sélectionner "CA certificate" |

### 💡 Remarque

> Un certificat CA de labo **augmente la capacité d'observation mais réduit la sécurité** de l'environnement. Son usage doit être :
> - **Temporaire** : retiré à la fin de la séance
> - **Documenté** : noté dans la trace d'audit
> - **Limité** : uniquement sur l'émulateur de labo, jamais sur un device personnel

---

## 🔵 Étape 9 — Produire un mini-rapport (preuve + contexte)

### Objectif
Documenter les observations de manière structurée, reproductible et professionnelle.

### Modèle de fiche de trace (à compléter)

---

#### 📄 FICHE DE TRACE — LAB 3 HTTP(S) ANDROID

**Date / Heure** : ___/___/202_ à ___h___  
**Opérateur** : _______________  
**Formateur / Module** : _______________

---

**1. PÉRIMÈTRE**

| Champ | Valeur |
|-------|--------|
| Environnement | Émulateur Android de labo (Android Studio AVD) |
| Cible testée | *(URL de la cible autorisée)* |
| Autorisation | *(Référence ou confirmation du formateur)* |
| Données sensibles | Aucune — environnement de test uniquement |

---

**2. CONFIGURATION**

| Paramètre | Valeur |
|-----------|--------|
| Burp Suite version | _____ |
| `<IP_HOTE>` | ___.___.___.___ |
| `<PORT_PROXY>` | _____ |
| Émulateur | Pixel X API XX |
| Certificat CA installé | ☐ Non / ☐ Oui (temporaire, retiré en fin de séance) |

---

**3. PREUVES**

> *(Insérer ici les captures d'écran de HTTP history et des requêtes observées)*

**Capture 1** : Vue HTTP history (liste des requêtes capturées)  
`[Coller capture ici]`

**Capture 2** : Détail d'une requête — vue Raw  
`[Coller capture ici]`

**Capture 3** : Détail d'une requête — panneau Inspector  
`[Coller capture ici]`

---

**4. ANALYSE**

**Données observées en transit :**

| Donnée | Localisation | Risque identifié |
|--------|-------------|-----------------|
| Paramètre `q=...` | URL (query string) | Exposé dans les logs serveur |
| Cookie `session_id` | Header Cookie | Absence d'attribut `HttpOnly` constatée |
| *(ajouter d'autres)* | | |

**Observations :**
- *(Ce qui a été observé, sans interprétation excessive)*
- Exemple : "Le paramètre de recherche est transmis en clair dans l'URL."

---

**5. RECOMMANDATIONS DÉFENSIVES**

| # | Recommandation | Priorité |
|---|---------------|----------|
| 1 | Éviter de transmettre des données sensibles dans les paramètres URL | Haute |
| 2 | Ajouter les attributs `HttpOnly`, `Secure`, `SameSite` aux cookies de session | Haute |
| 3 | Implémenter HTTPS avec HSTS côté serveur | Haute |
| 4 | Minimiser les données envoyées (principe de minimisation) | Moyenne |
| 5 | Appliquer les recommandations OWASP MASVS Network (MSTG-NETWORK-1/2) | Haute |

---

**6. STATUT DE NETTOYAGE**

- [ ] Proxy Android remis en "None"
- [ ] Certificat CA de labo retiré (si installé)
- [ ] Projet Burp fermé / archivé
- [ ] Aucune donnée sensible conservée

---

### Bonnes pratiques rédactionnelles

| ✅ Faire | ❌ Ne pas faire |
|---------|----------------|
| Séparer observation / hypothèse / recommandation | Affirmer sans preuve |
| Contextualiser chaque capture (date, cible, version) | Insérer des captures sans légende |
| Utiliser des tableaux pour les données structurées | Rédiger des paragraphes denses sans structure |
| Citer les standards (OWASP MASVS, MSTG) | Inventer des noms de vulnérabilités |

### Pourquoi ?

> **Une compétence d'audit se mesure à la qualité de la documentation**, pas au nombre d'outils utilisés. Un rapport reproductible — qu'une autre personne peut rejouer dans le même labo — a une valeur professionnelle réelle.

---

## ✔️ Checkpoints de validation

Avant de clore la séance, vérifier chaque point :

```
□ 1. Burp capture au moins une requête dans HTTP history
□ 2. Proxy listener actif, adresse et port documentés
□ 3. Proxy Android configuré en "Manuel" avec <IP_HOTE>:<PORT_PROXY>
□ 4. Intercept utilisé seulement pour démonstration, puis désactivé
□ 5. Au moins une requête analysée (méthode + headers + cookies)
□ 6. Rapport court produit (périmètre + config + preuves + analyse)
□ 7. Nettoyage planifié et effectué
```

---

## 🧹 Fin de lab — Nettoyage (hygiène)

### Procédure de nettoyage obligatoire

#### 1. Désactiver le proxy Android
1. Ouvrir **Paramètres → Wi-Fi** sur l'émulateur.
2. Modifier le réseau actif.
3. Remettre le proxy sur **"None"** (aucun proxy).
4. Enregistrer.

#### 2. Retirer le certificat CA (si installé)
1. Ouvrir **Paramètres → Sécurité → Chiffrement et identifiants**.
2. Aller dans **"Certificats de confiance" → "Utilisateur"**.
3. Localiser le certificat Burp PortSwigger.
4. Appuyer sur **"Supprimer"** et confirmer.

#### 3. Fermer et archiver Burp Suite
1. Fermer le projet temporaire Burp (ne pas sauvegarder de données sensibles).
2. Si des captures d'écran sont nécessaires pour le rapport, les exporter sans données personnelles.

#### 4. Vérification finale

```
□ Proxy Android remis en "None"
□ Certificat CA supprimé de l'émulateur
□ Projet Burp Suite fermé
□ Aucune donnée personnelle/sensible conservée
□ Émulateur revenu à un état neutre
```

### Pourquoi ?

> Un lab propre **évite de contaminer les séances suivantes** et réduit les risques de mauvaise utilisation accidentelle. Un certificat CA oublié sur un émulateur partagé est un vecteur de risque pour les prochains utilisateurs.

---

## 📖 Glossaire

| Terme | Définition |
|-------|-----------|
| **Proxy HTTP** | Intermédiaire réseau entre un client et un serveur, capable d'observer ou modifier les requêtes |
| **Listener** | Service en écoute sur un port réseau, attendant des connexions entrantes |
| **HTTP History** | Journal chronologique de toutes les requêtes capturées par Burp |
| **Intercept** | Mode bloquant de Burp : chaque requête est mise en attente avant transmission |
| **Certificate Authority (CA)** | Entité qui émet des certificats TLS reconnus comme de confiance |
| **TLS/HTTPS** | Protocole de chiffrement des communications HTTP (remplace HTTP en clair) |
| **Cookie** | Données stockées côté client, envoyées automatiquement avec chaque requête au domaine |
| **HttpOnly** | Attribut de cookie empêchant l'accès via JavaScript (protection XSS) |
| **Secure** | Attribut de cookie forçant l'envoi uniquement via HTTPS |
| **SameSite** | Attribut de cookie limitant l'envoi en contexte cross-site (protection CSRF) |
| **User-Agent** | En-tête HTTP identifiant le client (navigateur, OS, version) |
| **OWASP MASVS** | Mobile Application Security Verification Standard — référentiel de sécurité mobile |
| **ADB** | Android Debug Bridge — outil CLI pour interagir avec un émulateur/device Android |

---

## 📚 Ressources complémentaires

### Documentation officielle
- [Burp Suite Community — Documentation](https://portswigger.net/burp/documentation)
- [Proxy Listener Configuration — Burp](https://portswigger.net/burp/documentation/desktop/proxy/configure-browser)
- [Android Emulator — Android Studio](https://developer.android.com/studio/run/emulator)

### Standards de sécurité
- [OWASP MASVS (Mobile Application Security Verification Standard)](https://mas.owasp.org/MASVS/)
- [OWASP MSTG — Network Communication Testing](https://mas.owasp.org/MASTG/tests/android/MASVS-NETWORK/)
- [OWASP Top 10 Mobile](https://owasp.org/www-project-mobile-top-10/)

### Références HTTP et sécurité
- [MDN Web Docs — HTTP Headers](https://developer.mozilla.org/fr/docs/Web/HTTP/Headers)
- [MDN Web Docs — Cookies (Set-Cookie)](https://developer.mozilla.org/fr/docs/Web/HTTP/Headers/Set-Cookie)
- [RFC 9110 — HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)

### Pour aller plus loin
- [PortSwigger Web Security Academy (gratuit)](https://portswigger.net/web-security)
- [OWASP Android Security Testing Guide](https://mas.owasp.org/MASTG/Android/)

---

## 📁 Structure du dossier de lab

```
lab3-burpsuite-android/
├── README.md                    ← Ce fichier (documentation complète)
├── rapport/
│   ├── fiche_trace_lab3.md      ← Rapport d'audit à compléter
│   └── captures/                ← Captures d'écran documentées
│       ├── http_history.png
│       ├── requete_raw.png
│       └── inspector_view.png
└── ressources/
    └── burp_lab_ca.der          ← Certificat CA Burp (temporaire, à supprimer)
```

---

<div align="center">

---

**LAB 3 — Sécurité Mobile · Observation du trafic HTTP(S)**  
*Document pédagogique — Usage strictement limité au cadre du laboratoire*

📌 **Rappel final** : Retirer le certificat CA et désactiver le proxy à la fin de chaque séance.

---

</div>
