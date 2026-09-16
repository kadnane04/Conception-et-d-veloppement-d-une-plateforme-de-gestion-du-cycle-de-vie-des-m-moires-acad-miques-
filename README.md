# Plateforme GCVM — cycle de vie complet des mémoires

Gestion du cycle de vie des mémoires académiques — **Lomé Business School**

Cette livraison couvre **la totalité du cycle**, de la soumission du thème à l'archivage en bibliothèque numérique, pour les six profils concernés :

| Espace | Ce qu'il fait |
|---|---|
| **Administrateur** | Comptes, rôles, référentiels, journal d'activité |
| **Étudiant** | Thème, encadreur, dépôt versionné, version finale |
| **Responsable des Études** | Validation des thèmes, transmission, soutenances, archivage |
| **Encadreur** | Demandes d'encadrement, validation des mémoires |
| **Responsable Académique** | Validation ou rejet motivé des mémoires transmis |
| **Jury** | Invitations, consultation, observations, évaluation |

Les modules Administrateur et Étudiant **n'ont pas été modifiés dans leur
fonctionnement** : la phase 3 leur a ajouté des écrans (bibliothèque, ma
soutenance, dépôt de la version finale) sans en retoucher un seul comportement
déjà validé. Les 140 tests des deux premières phases passent tels quels.

---

## 1. Démarrer en cinq commandes

Sous Windows 10 / 11, dans l'invite de commandes ou PowerShell :

```bat
cd GCVM_Admin
python -m venv venv
venv\Scripts\activate
pip install -r requirements.txt
python manage.py migrate
python manage.py runserver
```

> **Raccourci :** `installer.bat` puis `demarrer.bat` font la même chose
> (`./installer.sh` et `./demarrer.sh` sous Linux/macOS).

### Compte administrateur initial

Il est créé automatiquement au premier `migrate`. Sans lui, personne ne
pourrait se connecter, puisqu'il n'existe pas d'inscription publique.

| | |
|---|---|
| Email | `admin@lbs.com` |
| Mot de passe | `admin1234567890` |
| Nom / Prénom | Admin / Admin |
| Fonction | Administrateur |
| Date de création | date du premier lancement |

Le mot de passe est **haché par Django** (PBKDF2-SHA256) : il n'est stocké en
clair nulle part. Ses valeurs se changent dans `.env` avant le premier
`migrate` (`ADMIN_INITIAL_EMAIL`, `ADMIN_INITIAL_MOT_DE_PASSE`).

### Données de démonstration (facultatif)

```bash
python manage.py seed_demo
python manage.py seed_cycle
```

## 2. Technologies

| Composant | Choix | Version |
|---|---|---|
| Langage | Python | 3.10 à 3.14 — **vérifié sur 3.14.3** |
| Framework | Django | 5.1 à 6.x, choisi automatiquement par pip |
| Base de données | SQLite | intégrée à Python (3.50 sur Python 3.14) |
| Base de données (plus tard) | PostgreSQL | 14 ou supérieur |
| Interface | HTML5, CSS3, JavaScript, **Bootstrap 5.3** | embarqué, aucun CDN |
| Icônes | Bootstrap Icons 1.11 | embarquées |
| Système | Windows 10 / 11, Linux, macOS | — |

**Deux dépendances seulement** : Django et `python-dotenv`. Toutes deux
s'installent en roue précompilée sur Python 3.14 — aucune compilation, aucun
outil système à installer.

Bootstrap et ses icônes sont livrés dans `static/vendor/` : l'application
fonctionne **sans connexion internet**, ce qui compte le jour de la soutenance.

---

## 3. Le circuit complet

C'est le cœur de la phase 3. Un mémoire ne change jamais d'état tout seul :
chaque transition est une décision prise par quelqu'un, tracée, et suivie de
notifications.

```
Étudiant ──soumet un thème──▶ Responsable des Études ──valide──▶ Étudiant
   │
   └──choisit un encadreur──▶ Encadreur ──accepte──▶ Responsable des Études (informé)
          │
          └──dépôt du mémoire──▶ Encadreur ──valide──▶ Responsable des Études
                                                              │
                                                     transmet │
                                                              ▼
                                          Responsable Académique ──valide──▶
                                                              │
                                     Responsable des Études ──planifie la soutenance
                                                              │
                                              Jurys invités ──acceptent (tous)
                                                              ▼
                                                        SOUTENANCE
                                                              │
                              Étudiant ──dépose la version finale──▶ Encadreur
                                                              │
                                        Responsable des Études ──▶ ARCHIVAGE
                                                              ▼
                                                  Bibliothèque numérique
```

À tout moment un rejet renvoie le dossier à l'étudiant **avec un motif
obligatoire** : la plateforme refuse un rejet sans motif, côté thème comme côté
encadreur comme côté responsable académique.

### Ce qui garantit la cohérence

Les transitions ne sont pas écrites dans les vues mais dans
`apps/memories/services.py` : une fonction par transition, qui fait les trois
choses ensemble — changer le statut, écrire la décision dans `DecisionMemoire`,
envoyer les notifications. Il devient impossible qu'un statut change sans que
la traçabilité et l'information suivent.

L'invariant « une soutenance est confirmée quand tous les jurys nécessaires ont
accepté » vit dans le modèle (`Soutenance.rafraichir_statut()`), pas dans une
vue : il s'applique quel que soit le chemin emprunté.

---

## 4. Les espaces, un par un

### 4.1 Administrateur (inchangé)

Connexion par email, mot de passe oublié en quatre étapes, tableau de bord à
cartes, création des quatre types de comptes, gestion par catégorie avec
tableau paginé, recherche, filtres, tri et six actions (consulter, modifier,
réinitialiser le mot de passe, désactiver, réactiver, supprimer), journal
d'activité, référentiels (catégories de thèmes, années académiques, guide de
rédaction).

> **Correction apportée en phase 3 :** le menu « … » des tableaux de gestion
> s'ouvre désormais dans toutes les situations. Voir la section 7.

### 4.2 Étudiant

Tableau de bord avec parcours et résumé général, choix du thème avec détection
automatique des doublons, choix de l'encadreur, dépôt versionné du mémoire,
notifications, profil. La phase 3 y ajoute :

- **Ma soutenance** — date, heure, salle et composition du jury, dès que tous
  les membres ont accepté ;
- **Dépôt de la version finale** après la soutenance, avec son propre circuit
  de validation ;
- **Bibliothèque** — consultation des mémoires archivés.

### 4.3 Responsable des Études

Le poste le plus chargé du circuit. Son tableau de bord montre, en cartes :
thèmes en attente, demandes d'encadrement acceptées, mémoires validés par les
encadreurs, mémoires à transmettre, soutenances à planifier, mémoires archivés.

- **Validation des thèmes** — la fiche montre le thème complet *et les thèmes
  proches déjà en base*, pour décider en connaissance de cause. Rejet = motif
  obligatoire. Notification « Nouveau thème soumis » à chaque dépôt.
- **Encadrements** — à l'acceptation d'un enseignant, notification
  « L'enseignant X accepte d'encadrer l'étudiant Y sur le thème Z ».
- **Réception des mémoires** — notification « Mémoire prêt pour transmission
  académique », avec Télécharger, Consulter et Transmettre.
- **Planification des soutenances** — date, heure, salle, plusieurs jurys.
  L'encadreur est membre de droit. La plateforme refuse deux soutenances qui se
  chevauchent dans la même salle.
- **Remplacement d'un jury** qui a refusé, sans repartir de zéro.
- **Archivage** de la version finale validée, et **import manuel** des anciens
  mémoires LBS (titre, auteur, année, filière, résumé, mots-clés, PDF).

### 4.4 Encadreur

Un enseignant se connecte à **son compte d'enseignant** — il n'existe pas de
compte « encadreur ». Son tableau de bord montre ses étudiants encadrés, les
demandes en attente, les mémoires à valider et ses soutenances à venir, avec
son quota d'encadrement.

Les demandes affichent l'étudiant, le thème, sa description et ses objectifs.
Accepter ou refuser notifie automatiquement l'étudiant ; en cas de refus,
l'étudiant peut choisir un autre encadreur. La validation des mémoires offre
Télécharger, Consulter, Valider, Rejeter — **motif obligatoire au rejet**.
L'encadreur valide aussi la **version finale** après la soutenance.

### 4.5 Responsable Académique

Tableau de bord : mémoires reçus, validés, rejetés. Chaque dossier montre
l'étudiant, l'encadreur, le thème et le PDF. Valider ou rejeter (motif
obligatoire) **notifie les trois parties** : l'étudiant, l'encadreur et le
responsable des études, motif compris.

### 4.6 Jury

**Aucun rôle « Jury » n'existe dans la plateforme, et c'est volontaire.** Les
jurys sont des enseignants existants : n'importe quel enseignant actif peut
être invité. La section « Mes soutenances » n'apparaît dans sa barre latérale
que s'il a effectivement été invité quelque part.

L'invitation « Vous avez été sélectionné comme membre du jury » montre
l'étudiant, le thème et le résumé — **pas le document**. Accepter ouvre l'accès
complet : téléchargement, consultation, observations, préparation. Refuser
libère la place pour un remplaçant. La soutenance n'est **confirmée que lorsque
tous les membres ont accepté** ; à ce moment-là l'étudiant et l'encadreur sont
notifiés avec la date, l'heure, la salle et la liste des jurys.

Après la soutenance, chaque membre saisit sa note, son commentaire et ses
recommandations ; la moyenne et la mention sont calculées.

### 4.7 Bibliothèque numérique

Ouverte à tous les profils connectés. Recherche plein texte sur le titre,
l'auteur, les mots-clés et l'encadreur ; filtres par année et par filière ; tri.
Chaque fiche montre titre, auteur, année, filière, résumé, mots-clés et
encadreur, avec consultation et téléchargement comptabilisés.

Une archive est une **fiche autonome** : elle recopie le titre, le nom de
l'auteur, celui de l'encadreur, l'année et la filière. Un mémoire archivé
survit donc à la suppression du compte de son auteur — ce qui est le propre
d'une bibliothèque.

### 4.8 Notifications

Un seul modèle générique pour toute la plateforme, avec titre, contenu, date et
état lu / non lu. Cloche dans la barre supérieure de **tous** les espaces, avec
le compteur de non-lus et l'aperçu des derniers messages ; page d'historique
complète avec filtre et « Tout marquer comme lu ».

---

## 5. Les décisions de conception à défendre

### 5.1 Un modèle utilisateur personnalisé, dès la première migration

L'identifiant de connexion est l'email, et les comptes portent des attributs
propres à l'établissement. Le `User` de Django ne convient pas. Poser un modèle
personnalisé **dès la première migration** évite une refonte ultérieure, qui
sous Django est une opération douloureuse.

### 5.2 Une table mère, une table par profil

`Utilisateur` porte ce qui est commun ; `ProfilEtudiant`, `ProfilProfesseur`,
`ProfilResponsableEtudes` et `ProfilResponsableAcademique` portent ce qui est
propre à chaque rôle, reliés par un `OneToOneField` dont la clé primaire est
celle de l'utilisateur.

C'est la traduction relationnelle de la généralisation du diagramme de classes.
Elle préserve simultanément deux choses qu'aucune autre stratégie ne préserve
ensemble :

- l'unicité de l'email sur l'ensemble des comptes ;
- les contraintes `NOT NULL` sur les attributs spécifiques (un étudiant a
  forcément une filière, un professeur a forcément un grade).

### 5.3 Encadreur et Jury ne sont pas des comptes — c'est le point à retenir

Le cahier des charges est explicite : *« Le professeur possède un seul compte.
Ne jamais créer plusieurs comptes pour la même personne. »*

Une sous-classe relationnelle serait permanente et exclusive : elle ne peut pas
représenter un professeur qui encadre un mémoire, siège au jury d'un autre, et
n'a aucun rôle sur un troisième.

La phase 3 confirme ce choix en le mettant à l'épreuve. Deux niveaux
coexistent :

| Niveau | Où il vit | Ce qu'il exprime |
|---|---|---|
| **Aptitude** | `ProfilProfesseur.statut` | ce que l'administrateur autorise |
| **Exercice** | `DemandeEncadrement`, `ParticipationJury` | ce que l'enseignant fait réellement, mémoire par mémoire et soutenance par soutenance |

L'espace Encadreur et l'espace Jury sont donc **deux vues du même compte** :
`yao.mensah@lbs.com` encadre Ama KOSSI et siège au jury de Kwami TETTEH depuis
une seule connexion. La garde d'accès au jury (`jury_requis`) interroge
`utilisateur.participations_jury.exists()` — une association, jamais un rôle.

**Formulation pour le mémoire :** *« Les classes Encadreur et MembreJury,
modélisées par généralisation dans le diagramme de classes, ont été
implémentées comme des rôles contextuels portés par des associations. Ce choix
permet à un même enseignant d'exercer plusieurs fonctions simultanément avec un
compte unique, et d'en conserver l'historique. »*

### 5.4 Les transitions dans une couche de service, pas dans les vues

Un changement de statut sans trace ni notification serait un bug invisible.
Regrouper les trois effets dans une seule fonction rend l'oubli impossible et
la règle lisible d'un coup d'œil — ce qui compte autant pour la soutenance que
pour la maintenance.

### 5.5 Les fichiers ne sont jamais servis directement

`django.conf.urls.static` a été retiré de `config/urls.py`. Chaque
téléchargement passe par une vue qui vérifie *qui demande quoi* avant de
renvoyer le fichier. Un jury qui n'a pas encore accepté reçoit une 404, pas le
PDF.

---

## 6. Sécurité

| Règle | Mise en œuvre |
|---|---|
| Aucune inscription publique | Aucune vue d'inscription n'existe. La racine du site mène à la connexion. |
| Comptes créés par l'administrateur seul | Les formulaires de création sont derrière le garde `administrateur_requis`. |
| Mots de passe hachés | `set_password()` / `make_password()` — PBKDF2-SHA256. Jamais de mot de passe en clair en base. |
| Aucun secret dans le code | `SECRET_KEY` et le mot de passe initial viennent de `.env`, absent du dépôt (`.gitignore`). `.env.example` sert de modèle. |
| Données fictives | Les comptes et documents de `seed_demo` / `seed_cycle` sont inventés : ni personnes, ni adresses, ni mots de passe réels. |
| Fichiers non publics | Aucun `MEDIA_URL` servi. Toute lecture passe par une vue qui contrôle le demandeur. |
| Extensions contrôlées | PDF seul pour les mémoires, taille maximale configurable, extension **et** type vérifiés. |
| Cloisonnement des espaces | Gardes centralisés dans `apps/accounts/gardes.py` ; un étudiant sur `/etudes/` est renvoyé vers `/etudiant/`. |
| Actions sensibles | Uniquement en POST, protégées par jeton CSRF, confirmées par l'interface. |
| Auto-protection | L'administrateur ne peut ni se désactiver ni se supprimer lui-même. |
| Anti-énumération | Connexion et mot de passe oublié répondent la même chose que le compte existe ou non. |
| Durcissement | HSTS, cookies sécurisés et redirection HTTPS s'activent automatiquement dès que `DEBUG=False`. |

---

## 7. La correction du menu « … »

Le symptôme signalé : les trois points s'affichaient, le menu ne se déroulait
pas. Les cinq causes possibles ont été traitées ensemble, plutôt que d'en
deviner une :

| Piste | Traitement |
|---|---|
| `z-index` | `.dropdown-menu { z-index: 1060; }` — au-dessus des en-têtes de tableau collants |
| `overflow: hidden` | menus marqués `data-menu-flottant`, positionnés par Popper en `strategy: "fixed"` : ils sortent du conteneur qui défile |
| `position: absolute` | remplacée par un positionnement fixe calculé sur la position réelle du bouton |
| Bootstrap dropdown | chaque bouton reçoit explicitement son instance au chargement, au lieu de dépendre de la délégation |
| JavaScript absent | **filet de sécurité maison** : si `bootstrap.Dropdown` n'existe pas, un gestionnaire léger prend le relais — ouverture, bascule vers le haut près du bas de l'écran, fermeture au clic extérieur, au défilement, au redimensionnement et à Échap |

Vérifié en conditions adverses, y compris en **bloquant volontairement le
chargement de `bootstrap.bundle.min.js`** : le menu s'ouvre, reste entièrement
dans l'écran (`position: fixed`, `z-index: 1060`) et rien n'est rogné.

Les six actions attendues sont bien là : **Consulter, Modifier, Réinitialiser le
mot de passe, Désactiver, Réactiver, Supprimer**. *Désactiver* et *Réactiver*
s'excluent l'une l'autre — le menu d'un compte actif propose la première, celui
d'un compte désactivé la seconde. Proposer les deux en même temps afficherait
toujours une action sans effet.

---

## 8. Structure du projet

```
GCVM_Admin/
├── config/                  settings, urls, wsgi, asgi
├── apps/
│   ├── accounts/            Utilisateur, 4 profils, authentification,
│   │                        mot de passe oublié, gardes, journal, jeux de démo
│   ├── administration/      MODULE ADMINISTRATEUR — tableau de bord, comptes,
│   │                        référentiels, documents institutionnels
│   │
│   │                        ── Données métier ──
│   ├── themes/              catégories, propositions, détection des doublons
│   ├── supervision/         demandes d'encadrement
│   ├── memories/            années, mémoires, versions, décisions, services
│   ├── defenses/            salles, soutenances, participations jury, évaluations
│   ├── archives/            mémoires archivés, consultations, bibliothèque
│   ├── notifications/       notifications internes et cloche
│   │
│   │                        ── Espaces (vues seules, aucun modèle) ──
│   ├── etudiant/            ESPACE ÉTUDIANT
│   ├── encadreur/           ESPACE ENCADREUR
│   ├── etudes/              ESPACE RESPONSABLE DES ÉTUDES
│   ├── academique/          ESPACE RESPONSABLE ACADÉMIQUE
│   └── jury/                ESPACE JURY
├── templates/
│   ├── base.html            squelette HTML
│   ├── base_app.html        sidebar + barre supérieure + cloche + messages
│   ├── comptes/             connexion, profil, mot de passe
│   ├── registration/        les 4 étapes du mot de passe oublié
│   ├── administration/      tableau de bord, gestion, référentiels, journal
│   ├── etudiant/ themes/ supervision/ memoires/
│   ├── etudes/ encadreur/ academique/ jury/ defenses/ archives/
│   ├── notifications/
│   └── partials/            sidebar, cartes, circuit, décisions, champ
├── static/
│   ├── css/gcvm.css         thème blanc / bleu / rouge
│   ├── js/gcvm.js           menus, confirmations, filtres, dépôt de fichier
│   └── vendor/              Bootstrap 5 et Bootstrap Icons (embarqués)
├── manage.py
├── requirements.txt
└── .env.example
```

La séparation est nette : les applications **métier** portent les modèles, les
applications **espace** ne portent que des vues et des gabarits. Un test le
vérifie, pour que la règle ne s'érode pas avec le temps.

---

## 9. Charte graphique

Strictement celle des phases 1 et 2, étendue aux nouveaux écrans :

- **Blanc dominant** : fond des pages, cartes, tableaux.
- **Nuances de bleu** : navigation, titres, actions courantes, étiquettes.
- **Rouge** : uniquement les actions destructrices et les alertes — désactiver,
  supprimer, rejeter, erreurs. Employé partout, il ne signalerait plus rien.
- Sidebar bleu nuit, avec pastilles de compteur par rôle ; tableaux aérés ;
  formulaires groupés par section ; frise de circuit sur les écrans de suivi.
- Responsive : la sidebar se replie en dessous de 992 px, les tableaux défilent
  horizontalement, la page de connexion passe sur une colonne.

---

## 10. Tests

```bash
python manage.py test
```

**201 tests**, tous au vert sous Python 3.14.3 / Django 6.1.1 :

| Module | Tests |
|---|---|
| Administrateur (`accounts`, `administration`) | 74 |
| Étudiant (`etudiant`) | 66 |
| Cycle complet (`etudes`) | 61 |

Les 140 tests des phases 1 et 2 sont **inchangés** et passent tels quels :
c'est la preuve de non-régression demandée.

| Domaine | Ce qui est vérifié |
|---|---|
| Modèle | email identifiant et unique, normalisation, hachage, propriétés de rôle |
| Professeur | encadreur seul, jury seul, cumul des deux, changement de statut sans second compte |
| Compte initial | présent après `migrate`, mot de passe haché et fonctionnel |
| Connexion | succès, échec, casse de l'email, compte désactivé, absence de page d'inscription |
| Routes | anonyme redirigé, étudiant tenu hors de l'administration, aiguillage par rôle |
| Mot de passe oublié | envoi, silence pour une adresse inconnue, compte désactivé, parcours complet |
| Création / gestion | compte + profil, unicité entre rôles, recherche, filtres, tri, pagination, six actions |
| Doublons | normalisation, titres proches détectés, titres distincts ignorés, avertissement puis confirmation |
| Thème | une proposition à la fois, non modifiable après soumission, reproposition après rejet |
| Encadreur | étape verrouillée sans thème validé, une seule demande en cours, quota |
| Mémoire | PDF seul, taille maximale, versionnage sans écrasement |
| **Cycle complet** | parcours du dépôt à l'archivage, statut par statut |
| **Motifs** | rejet sans motif refusé — thème, encadreur, responsable académique |
| **Notifications** | la validation académique informe bien l'étudiant, l'encadreur et le responsable des études |
| **Soutenance** | quorum, encadreur membre de droit, n'importe quel enseignant peut siéger, un refus bloque la confirmation, remplacement, conflit de salle, barème des mentions |
| **Jury** | document fermé avant acceptation, ouvert après, un seul compte pour encadrer et siéger |
| **Version finale** | dépôt après soutenance, validation encadreur puis études, archivage |
| **Bibliothèque** | visibilité par profil, recherche, compteurs, import manuel |
| **Cloisonnement** | chaque espace inaccessible aux autres rôles |
| Architecture | les applications d'espace restent sans modèle métier |

### Vérification sur l'environnement cible

Sur un interpréteur **CPython 3.14.3** : installation en roues précompilées
(Django 6.1.1, python-dotenv 1.2.3, asgiref 3.12.1, sqlparse 0.6.0), SQLite
3.50.4, `check` sans avertissement, migrations appliquées, **201 tests au
vert**, jeux de démonstration exécutés, serveur démarré, **58 pages parcourues
pour sept comptes sans une seule erreur serveur**, pages de détail atteintes
depuis chaque liste, et cloisonnement vérifié compte par compte (un étudiant
sur `/etudes/`, `/academique/` ou `/encadreur/` est systématiquement renvoyé
vers son propre espace).

---

## 11. Passer à PostgreSQL

Aucun code à modifier. Décommentez `psycopg2-binary` dans `requirements.txt`,
installez-le, puis dans `.env` :

```
DB_ENGINE=postgresql
DB_NAME=gcvm
DB_USER=gcvm
DB_PASSWORD=...
DB_HOST=localhost
DB_PORT=5432
```

Puis `python manage.py migrate`.

---

## 12. Ce qui vient ensuite

Le cycle est complet et fonctionne de bout en bout. Ce qui reste relève de
l'exploitation, pas de la conception :

1. **Envoi réel des emails** — renseigner `EMAIL_HOST` dans `.env` suffit ; le
   code n'a pas à changer. Les notifications internes pourraient alors être
   doublées par un email.
2. **Statistiques et exports** — taux de validation, délais moyens par étape,
   export PDF ou Excel des listes. Les données sont déjà là, dans
   `DecisionMemoire`.
3. **Dépôt en production** — PostgreSQL, `DEBUG=False`, serveur WSGI, sauvegarde
   des fichiers. La bascule de base est déjà prévue par `.env`.
4. **Détection de plagiat** — la normalisation des textes et le calcul de
   similarité utilisés pour les doublons de thèmes sont réutilisables sur le
   contenu des mémoires.
