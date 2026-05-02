  
**CAHIER DES CHARGES FONCTIONNEL**

Plateforme de Cours Particuliers

*Version 1.0 — Prototype*

| Version | 1.0 |
| :---- | :---- |
| **Statut** | Prêt pour développement |
| **Stack** | Next.js 15 · TypeScript · Node 22 LTS · PostgreSQL 16 (Neon) · Prisma 5 · Auth.js v5 · Stripe Connect Express · Google Calendar API · Resend · Cloudflare R2 · Vercel |
| **Objectif** | Livrer un prototype fonctionnel validant le modèle économique avec un suivi pédagogique gamifié différenciant |

# **0\. Introduction et objectif du produit**

Cette plateforme web permet à des professeurs particuliers de gérer leurs cours, leur agenda et leurs paiements. Les parents ou étudiants autonomes réservent et paient en avance sur la plateforme. Le professeur reçoit ses paiements directement sur son compte Stripe Connect au moment de la réservation (mode *destination charges*) ; le versement bancaire effectif suit le calendrier de payout standard de Stripe. La plateforme prélève une commission automatique configurable sur chaque transaction via `application_fee_amount`.

> **Note POC :** la version 1 ne gère PAS d'escrow applicatif (pas de hold 24h côté plateforme). Les statuts `EN_ATTENTE / REVERSE` du modèle `Payment` reflètent l'état applicatif, pas un blocage de fonds. Un vrai escrow nécessiterait *separate charges + transfers* et engage la plateforme sur le plan réglementaire — hors scope POC.

L'élève (mineur rattaché à un parent, ou adulte autonome) accède à un espace personnel contenant ses cours à venir, ses devoirs et ses ressources pédagogiques. L'accès à cet espace est conditionné à l'existence d'au moins un cours à venir.

| Philosophie v1 : livrer vite, valider le modèle. On évite toute complexité non indispensable. On délègue le maximum à des services tiers éprouvés (Stripe, Google, Resend). On ne code que ce qu'aucun service ne peut faire à notre place. |
| :---- |

## **0.1 Rôles utilisateurs**

| ADMIN | Accès total en lecture/écriture sur tous les utilisateurs, transactions et logs de connexion. |
| :---- | :---- |
| **PROF** | Gère son agenda, ses élèves, ses devoirs, son compte Stripe et sa politique de remboursement. |
| **PARENT** | Réserve et paie des cours pour ses enfants. Consulte le suivi pédagogique. |
| **ELEVE** | Accède à ses cours à venir, devoirs et ressources. Compte séparé du parent. |
| **ETUDIANT** | Adulte autonome. Réserve et paie lui-même. Dashboard mixte parent \+ élève. |

## **0.2 Principes techniques non-négociables**

* Tout montant d'argent est manipulé en centimes (entier) côté serveur. Jamais de float.

* Aucune donnée bancaire ne transite par nos serveurs. Stripe est seul responsable.

* La commission plateforme est définie dans la variable d'environnement PLATFORM\_FEE\_PERCENT (défaut : 10).

* Chaque API Route Next.js vérifie le rôle de l'utilisateur avant toute action (middleware Auth.js).

* Les tokens Google OAuth sont stockés chiffrés en base via Auth.js. Jamais en clair.

* Toutes les dates sont stockées en UTC en base. L'affichage est converti dans le fuseau du navigateur.

* Aucun fichier utilisateur n'est stocké en base de données. On stocke uniquement des URLs (Cloudflare R2).

* **RGPD — collecte minimale :** pas d'adresse IP stockée nulle part (ni dans SessionLog, ni dans les logs). Le user-agent navigateur peut être conservé pour la sécurité. Pas de nom/prénom obligatoire : l'identité publique est un `username` (pseudo) librement choisi par l'utilisateur. On ne collecte que ce qui est strictement nécessaire au service.

* **RGPD — consentement :** à l'inscription, l'utilisateur accepte des CGU courtes et claires. L'horodatage de l'acceptation est stocké dans `User.cguAcceptedAt` (preuve de consentement). Pas de tracking publicitaire, pas de partage de données à des tiers hors prestataires techniques (Stripe, Google, Resend, Cloudflare).

* **RGPD — mineurs :** la création d'un compte `ELEVE` par un parent vaut consentement parental. L'horodatage est stocké dans `ParentEleve.parentalConsentAt`. Un élève mineur ne peut pas s'inscrire seul.

# **1\. Architecture des routes (Next.js App Router)**

Toutes les routes protégées vérifient le rôle via le middleware Auth.js. Un utilisateur qui accède à une route non autorisée est redirigé vers /login avec un paramètre callbackUrl.

| / (page de garde) | Publique |
| :---- | :---- |
| **/login** | Publique |
| **/register** | Publique |
| **/admin/\*** | Rôle ADMIN uniquement |
| **/dashboard/prof/\*** | Rôle PROF uniquement |
| **/dashboard/parent/\*** | Rôle PARENT uniquement |
| **/dashboard/eleve/\*** | Rôle ELEVE uniquement |
| **/dashboard/etudiant/\*** | Rôle ETUDIANT uniquement |
| **/booking/\[profId\]** | Authentifié (PARENT ou ETUDIANT) |
| **/api/webhooks/stripe** | Publique — vérification signature Stripe obligatoire |
| **/api/webhooks/google** | Publique — vérification token Google obligatoire |

| 🌐  2\. Pages publiques *Page de garde, login, inscription* |
| :---- |

| PUB-01 | Page de garde |
| :---- | :---- |
| **Description** Page d'accueil publique de la plateforme. C'est la seule page visible sans être connecté avec du contenu marketing. **Comportement attendu** Affiche le nom de la plateforme en grand, centré, avec une accroche courte. Un seul bouton principal : 'Se connecter'. Ce bouton redirige vers /login. Un lien secondaire 'Créer un compte' redirige vers /register. La page est statique (Server Component). Aucun appel API au chargement. Si l'utilisateur est déjà connecté et accède à /, il est redirigé automatiquement vers son dashboard correspondant à son rôle. **Critères d'acceptation** Un utilisateur non connecté voit le bouton 'Se connecter'. Un utilisateur connecté est redirigé vers /dashboard/\[son-role\]. La page se charge en moins de 1 seconde (pas de JS client nécessaire). |  |

| PUB-02 | Page de login |
| :---- | :---- |
| **Description** Page permettant à tout utilisateur de s'identifier. Deux méthodes disponibles : email/mot de passe ou Google OAuth. **Comportement attendu** Formulaire avec deux champs : Email et Mot de passe. Bouton 'Se connecter avec Google' qui déclenche le flow OAuth Google via Auth.js. En cas d'erreur (mauvais mot de passe, email inconnu) : afficher un message d'erreur générique 'Email ou mot de passe incorrect' — ne pas préciser lequel est faux (sécurité). Après connexion réussie : rediriger vers le dashboard du rôle de l'utilisateur. Si callbackUrl est présent dans l'URL, rediriger vers cette URL à la place. Lien 'Mot de passe oublié' — non implémenté en v1, afficher 'Fonctionnalité bientôt disponible'. Lien 'Créer un compte' vers /register. **Critères d'acceptation** Email/mdp correct → redirection dashboard. Email/mdp incorrect → message d'erreur, pas de redirection. OAuth Google → si email existe déjà en base (compte email/mdp), lier les comptes. Si email inconnu, afficher 'Aucun compte trouvé avec cet email Google. Veuillez créer un compte.' Accès à /login si déjà connecté → redirection dashboard. |  |

| PUB-03 | Page de création de compte |
| :---- | :---- |
| **Description** Page d'inscription pour les nouveaux utilisateurs. Le rôle est choisi dès l'inscription. **Comportement attendu** Champs : **Pseudo (username)** — affiché publiquement, obligatoire, entre 3 et 30 caractères. Email. Mot de passe (min 8 caractères, 1 majuscule, 1 chiffre). Confirmation mot de passe. Sélecteur de rôle : Professeur / Parent / Élève / Étudiant autonome. Chaque option affiche une courte description en dessous. Si rôle ELEVE est sélectionné : afficher un champ supplémentaire 'Code d'invitation du parent' (optionnel). Si rempli, l'élève est automatiquement lié au parent correspondant. Bouton 'Créer mon compte avec Google' : pré-remplit email depuis Google, le pseudo et le rôle restent à remplir manuellement. Validation côté client ET côté serveur (API Route POST /api/auth/register). Après inscription : connexion automatique et redirection vers le dashboard du rôle. Si c'est un PROF, redirection vers l'onboarding /dashboard/prof/onboarding. Afficher un lien vers les CGU, cochable obligatoirement avant de valider. À la validation, l'API `POST /api/auth/register` enregistre `User.cguAcceptedAt = now()` (preuve de consentement RGPD). **Critères d'acceptation** Tous les champs obligatoires remplis \+ rôle sélectionné \+ CGU cochées → compte créé, connexion automatique. Email déjà existant → message 'Un compte existe déjà avec cet email'. Mots de passe non identiques → message d'erreur sur le champ confirmation. Mot de passe trop faible → message précis sur les critères manquants. Code d'invitation invalide → message 'Code d'invitation invalide', inscription quand même possible sans liaison. |  |

| 🎓  3\. Onboarding professeur *Étapes de configuration obligatoires à la première connexion* |
| :---- |

L'onboarding est un tunnel en 4 étapes affiché à la première connexion du prof. Tant que l'onboarding n'est pas terminé, toutes les routes /dashboard/prof/\* redirigent vers /dashboard/prof/onboarding. Une barre de progression (étape X/4) est affichée en haut.

| ONB-01 | Étape 1 — Informations personnelles |
| :---- | :---- |
| **Description** Le prof renseigne ses informations de base et professionnelles. **Comportement attendu** Champs : **Pseudo (username)** — pré-rempli depuis l'inscription, modifiable ici. Photo de profil (upload optionnel vers R2). Biographie courte (textarea, max 500 caractères). Matières enseignées (tags sélectionnables : Maths, Français, Anglais, Histoire-Géo, Physique-Chimie, SVT, Philosophie, Espagnol, Allemand, Informatique, Autre). Tarif horaire en euros (champ numérique, min 1, max 500). Ce tarif est celui affiché aux parents. Niveau(x) enseigné(s) : cases à cocher — Primaire, Collège, Lycée, Supérieur, Adulte. Bouton 'Suivant' enregistre et passe à l'étape 2\. **Critères d'acceptation** Tous les champs obligatoires (pseudo, au moins 1 matière, tarif, au moins 1 niveau) remplis → passage étape 2\. Tarif ≤ 0 ou \> 500 → message d'erreur. Photo uploadée : stockée dans R2, URL enregistrée en base. |  |

| ONB-02 | Étape 2 — Connexion Google Calendar |
| :---- | :---- |
| **Description** Le prof connecte son compte Google pour synchroniser son agenda. C'est obligatoire. **Comportement attendu** Afficher un bloc explicatif : 'Nous utilisons Google Calendar pour afficher vos disponibilités aux parents et créer automatiquement les événements de cours avec un lien Google Meet.' Bouton 'Connecter Google Calendar' déclenche le flow OAuth Google avec les scopes : openid, email, profile, https://www.googleapis.com/auth/calendar. Après autorisation Google : les tokens (access\_token, refresh\_token, expires\_at) sont stockés chiffrés en base liés au prof. Afficher une confirmation : 'Google Calendar connecté ✓ — Agenda : \[nom@gmail.com\]'. Bouton 'Suivant' actif uniquement si Google Calendar est connecté. **Critères d'acceptation** OAuth Google réussi → tokens stockés, bouton Suivant actif. OAuth Google refusé → message 'La connexion Google Calendar est obligatoire pour utiliser la plateforme.' Si le prof ferme la fenêtre OAuth sans autoriser → rester sur l'étape 2\. |  |

| ONB-03 | Étape 3 — Connexion Stripe |
| :---- | :---- |
| **Description** Le prof connecte son compte bancaire via Stripe Connect Express pour recevoir ses paiements. **Comportement attendu** Afficher un bloc explicatif : 'Stripe gère vos paiements de façon sécurisée. Vos données bancaires ne transitent jamais par notre plateforme. Vos paiements arrivent directement sur votre compte Stripe Connect au moment de la réservation, puis sont versés sur votre compte bancaire selon le calendrier de payout standard de Stripe.' Bouton 'Configurer mes paiements avec Stripe' : appel à l'API Route POST /api/stripe/connect/onboard qui génère un lien Stripe Connect Express Onboarding et redirige le prof vers ce lien. Stripe redirige vers /dashboard/prof/onboarding?stripe=success ou ?stripe=error. Si stripe=success : stocker le stripeAccountId en base, marquer l'étape comme complétée. Si stripe=error : afficher 'Une erreur s'est produite avec Stripe. Veuillez réessayer.' Bouton 'Suivant' actif uniquement si stripeAccountId est présent en base. **Critères d'acceptation** Onboarding Stripe terminé → stripeAccountId sauvegardé, bouton Suivant actif. Onboarding Stripe incomplet (prof n'a pas tout rempli chez Stripe) → bouton 'Reprendre la configuration Stripe' qui relance le lien. |  |

| ONB-04 | Étape 4 — Politique de remboursement |
| :---- | :---- |
| **Description** Le prof définit sa politique de remboursement. Cette politique sera affichée aux parents avant qu'ils paient. **Comportement attendu** Quatre options radio :   • 'Remboursement intégral si annulation plus de 2h avant le cours'   • 'Remboursement intégral si annulation plus de 24h avant le cours'   • 'Remboursement intégral si annulation plus de 48h avant le cours'   • 'Pas de remboursement (sauf cas de force majeure décidé manuellement)' Champ texte libre optionnel 'Précisions supplémentaires' (max 300 caractères). Bouton 'Terminer la configuration' → marque l'onboarding comme terminé (champ onboardingCompleted \= true en base), redirige vers /dashboard/prof. **Critères d'acceptation** Une option radio sélectionnée → bouton actif. Aucune option → bouton désactivé. Après clic 'Terminer' → redirection vers le dashboard prof principal. |  |

| 🔴  4\. Dashboard Administrateur *Vue globale de la plateforme, utilisateurs et transactions* |
| :---- |

| ADM-01 | Page principale admin — Liste des utilisateurs |
| :---- | :---- |
| **Description** Vue tabulaire de tous les utilisateurs inscrits sur la plateforme. **Comportement attendu** Tableau avec colonnes : ID, **Pseudo (username)**, Email, Rôle (badge coloré), Date d'inscription, Dernière connexion, Statut (Actif / Banni). Filtres : par rôle (dropdown), par statut, barre de recherche par pseudo/email. Pagination : 25 utilisateurs par page. Clic sur une ligne → ouvre une modale avec le détail complet de l'utilisateur. Dans la modale : bouton 'Bannir cet utilisateur' (confirmation requise) et 'Voir ses transactions'. **Critères d'acceptation** La liste charge les utilisateurs depuis la base via Server Component. Filtre par rôle 'PROF' → n'affiche que les profs. Bannir un utilisateur → champ bannedAt rempli en base, l'utilisateur ne peut plus se connecter (AuthJS vérifie ce champ au login). |  |

| ADM-02 | Page logs de connexion |
| :---- | :---- |
| **Description** Historique de toutes les connexions à la plateforme. **Comportement attendu** Tableau avec colonnes : Utilisateur (pseudo \+ email), Rôle, Date/heure de connexion, User-Agent (navigateur uniquement — **pas d'adresse IP, conformité RGPD**). Chaque connexion réussie est enregistrée en base (table SessionLog). Filtre par utilisateur (recherche pseudo/email) et par plage de dates. Pagination : 50 entrées par page. **Critères d'acceptation** À chaque connexion réussie via Auth.js (callback signIn) → insertion d'une entrée SessionLog sans IP. L'admin voit toutes les entrées sans restriction. |  |

| ADM-03 | Page transactions |
| :---- | :---- |
| **Description** Vue globale de tous les paiements, reversements et commissions. **Comportement attendu** Tableau avec colonnes : Date, Parent/Étudiant, Prof, Élève, Montant brut, Commission plateforme, Montant reversé, Statut (EN\_ATTENTE / REVERSE / REMBOURSE). Filtre par statut et par plage de dates. Total des commissions générées affiché en haut de page. **Critères d'acceptation** Les données viennent de la table Payment en base. Statut EN\_ATTENTE → paiement encaissé, cours pas encore passé ou \< 24h. Statut REVERSE → transfer Stripe effectué au prof. Statut REMBOURSE → remboursement Stripe effectué au parent. |  |

| 🟡  5\. Dashboard Professeur *Gestion des cours, élèves, disponibilités et paiements* |
| :---- |

| PROF-01 | Page d'accueil dashboard prof |
| :---- | :---- |
| **Description** Vue synthétique des informations clés du professeur. **Comportement attendu** Bloc 'Prochain cours' : affiche le prochain cours à venir (date, heure, élève, lien Google Meet). Bloc 'Revenus du mois' : total des reversements reçus sur le mois courant. Bloc 'Élèves actifs' : nombre d'élèves avec au moins un cours dans les 30 derniers jours. Bloc 'Devoirs en attente de correction' : nombre de devoirs rendus par des élèves non encore corrigés. Raccourcis rapides : 'Voir mes disponibilités', 'Voir mes élèves', 'Donner un devoir'. **Critères d'acceptation** Si aucun cours à venir → afficher 'Aucun cours prévu. Vérifiez vos disponibilités.' Tous les blocs sont des Server Components avec données fraîches à chaque chargement. |  |

| PROF-02 | Page informations personnelles |
| :---- | :---- |
| **Description** Le prof consulte et modifie ses informations de profil. **Comportement attendu** Formulaire pré-rempli avec les données actuelles : **Pseudo (username)**, Email (non modifiable), Photo de profil, Biographie, Matières, Niveaux, Tarif horaire. Bouton 'Sauvegarder les modifications' → appel PATCH /api/prof/profile. Section séparée 'Politique de remboursement' : affiche la politique actuelle avec un bouton 'Modifier' qui ouvre le même formulaire que ONB-04. Section 'Compte Stripe' : affiche 'Compte Stripe connecté ✓' et un bouton 'Gérer mon compte Stripe' qui redirige vers le Stripe Express Dashboard (lien généré via l'API Stripe). Section 'Google Calendar' : affiche l'email Google connecté et un bouton 'Déconnecter / Reconnecter'. **Critères d'acceptation** Modification tarif → nouvelle valeur en base, applicable aux nouveaux bookings uniquement (pas rétroactif). Photo modifiée → ancienne URL R2 supprimée, nouvelle URL stockée. |  |

| PROF-03 | Page disponibilités |
| :---- | :---- |
| **Description** Le prof définit ses créneaux horaires disponibles pour les réservations. **Comportement attendu** Interface visuelle : sélecteur de jours de la semaine (Lundi → Dimanche) avec pour chaque jour actif, des plages horaires (heure début / heure fin) ajoutables. Exemple : Lundi 14h00–18h00, Mercredi 09h00–12h00. Durée par défaut d'un créneau : 1h (configurable à 30min, 1h, 1h30, 2h). Bouton 'Sauvegarder' → appel POST /api/prof/availability qui remplace en transaction toutes les `ProfAvailability` du prof. Aucune écriture dans Google Calendar pour les dispos (Google Calendar n'est utilisé que pour les événements de cours réels et leurs liens Meet). Afficher aussi un aperçu en lecture seule du calendrier de la semaine en cours avec les créneaux déjà réservés (bookings) affichés en orange. Bouton 'Voir dans Google Calendar' ouvre le Google Calendar du prof dans un nouvel onglet. **Critères d'acceptation** Sauvegarde réussie → confirmation 'Disponibilités mises à jour'. Aucune dépendance Google sur cette sauvegarde. Un créneau déjà réservé ne peut pas être supprimé. |  |

| PROF-04 | Page élèves |
| :---- | :---- |
| **Description** Liste de tous les élèves du professeur avec accès à leur fiche. **Comportement attendu** Liste avec pour chaque élève : photo (initiales du pseudo si pas de photo), **Pseudo (username)**, Niveaux, Matière principale, Nombre de cours donnés, Prochain cours. Barre de recherche par pseudo. Clic sur un élève → ouvre la fiche élève. Fiche élève contient : Informations de base (pseudo, niveau), Historique des cours (liste avec date, durée, notes post-séance du prof), Suivi scores (agrégat des évaluations post-séance), Devoirs assignés et leur statut, Ressources partagées. **Critères d'acceptation** Si le prof n'a pas encore d'élèves → afficher 'Aucun élève pour l'instant. Les élèves apparaîtront ici une fois leurs cours réservés.' La fiche élève est en lecture seule ici (sauf pour ajouter des notes). |  |

| PROF-05 | Page clients |
| :---- | :---- |
| **Description** Liste de tous les parents et étudiants autonomes liés au professeur. **Comportement attendu** Deux onglets : 'Parents' et 'Étudiants autonomes'. Pour chaque client : **Pseudo (username)**, Email, Nombre de cours achetés, Montant total payé, Enfants associés (pour les parents). Clic sur un client → modale avec le détail et l'historique des bookings. **Critères d'acceptation** Un client apparaît dès qu'il a effectué au moins un booking payé avec ce prof. |  |

| PROF-06 | Page devoirs |
| :---- | :---- |
| **Description** Le prof crée et assigne des devoirs ou exercices à ses élèves. **Comportement attendu** Formulaire de création : Titre du devoir, Élève cible (dropdown de ses élèves), Cours associé (dropdown des cours passés ou à venir de cet élève), Description/consigne (textarea riche avec formatage gras/italique/liste), Date limite de rendu, Pièces jointes optionnelles (upload vers R2, max 3 fichiers de 10 Mo chacun). Liste des devoirs existants avec colonnes : Titre, Élève, Cours, Date limite, Statut (NON\_RENDU / RENDU / CORRIGE). Quand un élève a rendu un devoir : un badge 'À corriger' apparaît. Le prof peut cliquer pour voir le rendu et écrire un commentaire de correction. Bouton 'Ajouter une ressource' : upload d'un fichier ou ajout d'un lien URL avec un titre, visible dans 'Mes ressources' de l'élève. **Critères d'acceptation** Devoir créé → visible dans le dashboard de l'élève ciblé. Élève rend le devoir → statut passe à RENDU, badge 'À corriger' apparaît pour le prof. Prof écrit correction → statut passe à CORRIGE, l'élève voit la correction. |  |

| PROF-07 | Questionnaire post-séance professeur |
| :---- | :---- |
| **Description** Après chaque cours, le prof est invité à remplir un court questionnaire d'évaluation. Ce questionnaire s'affiche automatiquement 15 minutes après l'heure de fin du cours. **Comportement attendu** Déclenchement : une notification apparaît dans le dashboard du prof '⏰ Votre cours avec \[Élève\] s'est terminé. Comment s'est-il passé ?'. Formulaire : **Présence de l'élève** : case à cocher 'Présent / Absent' (obligatoire — alimente `PostSeanceProf.attendance`). Puis 3 questions à 5 niveaux représentés par des **icônes Lucide** (pas d'emoji) :   • Attitude de l'élève (1 à 5)   • Niveau de compréhension (1 à 5)   • Confiance dans la matière (1 à 5)   • A-t-il besoin d'une réunion avec le parent ? : Oui / Non (boutons) Champ texte libre 'Remarques pour l'élève' (optionnel, max 1000 caractères) — visible par le parent. Champ texte libre 'Notes privées' (optionnel, max 1000 caractères) — visible uniquement par le prof. Bouton 'Enregistrer' → sauvegarde en base, la notification disparaît. Bouton 'Passer' → le questionnaire est marqué comme ignoré, disparaît. Si 'besoin réunion \= Oui' : un email automatique est envoyé au parent via Resend pour l'informer. **Critères d'acceptation** Le questionnaire n'apparaît que pour les cours dont le statut est TERMINE. Un seul questionnaire par cours et par prof. Si ignoré → aucune relance (pas de spam). Les réponses sont visibles dans la fiche élève et dans le dashboard parent. |  |

| PROF-08 | Génération du code d'invitation parent |
| :---- | :---- |
| **Description** Le prof peut générer un code unique à partager avec un parent pour que ce parent soit lié à lui. **Comportement attendu** Sur la page clients : bouton 'Générer un lien d'invitation'. Génère un code unique (UUID court) valable 7 jours. Le lien généré est de la forme : https://\[domaine\]/join/\[code\]. Le prof peut copier le lien avec un bouton 'Copier le lien'. Quand un parent accède à ce lien et est connecté (ou se connecte) → il est automatiquement lié au prof. **Critères d'acceptation** Lien valide → parent lié au prof, confirmation affichée. Lien expiré (\> 7 jours) → message 'Ce lien d'invitation a expiré. Demandez un nouveau lien au professeur.' Lien déjà utilisé → message 'Vous êtes déjà lié à ce professeur.' |  |

| 🟢  6\. Dashboard Parent *Réservation, paiement et suivi pédagogique* |
| :---- |

| PAR-01 | Page d'accueil dashboard parent |
| :---- | :---- |
| **Description** Vue synthétique pour le parent. **Comportement attendu** Bloc 'Prochains cours' : liste des 3 prochains cours de ses enfants (date, heure, élève, prof). Bloc 'Paiements récents' : les 3 dernières transactions. Section 'Mes profs' avec un bouton 'Ajouter un professeur' (qui renvoie vers PAR-02). |  |

| PAR-02 | Section — Mes professeurs |
| :---- | :---- |
| **Description** Le parent gère ses professeurs associés. **Comportement attendu** Liste des profs déjà associés : photo, nom, matières, tarif horaire, bouton 'Réserver un cours' et bouton 'Voir les disponibilités'. Bouton 'Ajouter un professeur' : affiche un champ texte pour saisir un code ou lien d'invitation. En v1, seule cette méthode est disponible (pas de moteur de recherche). Après saisie d'un code valide : le prof apparaît dans la liste. **Critères d'acceptation** Code valide → prof ajouté, confirmation. Code invalide → message d'erreur. Prof déjà dans la liste → 'Ce professeur est déjà dans votre liste.' |  |

| PAR-03 | Page de réservation d'un cours |
| :---- | :---- |
| **Description** Le parent réserve un ou plusieurs créneaux auprès d'un professeur associé. **Comportement attendu** URL : /booking/\[profId\] — accessible uniquement si le parent est lié au prof. Afficher : photo \+ nom du prof, matières, tarif horaire, politique de remboursement (texte complet). Calendrier visuel (vue semaine) : affiche les créneaux disponibles du prof (récupérés depuis Google Calendar via l'API). Les créneaux déjà réservés sont grisés et non cliquables. Le parent sélectionne un ou plusieurs créneaux (chacun \= 1 cours). Chaque sélection est mise en évidence. Résumé de la commande : liste des cours sélectionnés avec date/heure, durée, prix unitaire, sous-total, commission affichée (non), total à payer. Sélecteur 'Pour quel élève ?' : dropdown avec les enfants du parent. Si un seul enfant, pré-sélectionné automatiquement. Bouton 'Procéder au paiement' → redirige vers la page de paiement Stripe (Stripe Checkout). **Critères d'acceptation** Créneau disponible cliquable → sélectionné, ajouté au récapitulatif. Créneau déjà réservé → grisé, non cliquable, tooltip 'Ce créneau est déjà pris'. Au moins 1 créneau sélectionné ET élève choisi → bouton paiement actif. Aucun créneau → bouton désactivé. Après paiement réussi → redirection vers /dashboard/parent/paiements avec message de confirmation. *⚠️  Note : La commission plateforme n'est pas affichée au parent. Le tarif affiché est le tarif du prof.* |  |

| PAR-04 | Section — Mes enfants |
| :---- | :---- |
| **Description** Le parent consulte le profil, le suivi pédagogique et la progression gamifiée de ses enfants. **Comportement attendu** Liste des enfants liés au compte parent. Bouton 'Ajouter un enfant' : affiche un formulaire (**Pseudo**, Niveau scolaire, Email optionnel). Crée un compte ELEVE et envoie un email d'invitation à l'élève (si email fourni). Clic sur un enfant → fiche de suivi de l'enfant contenant :   • Informations de base (pseudo, niveau).   • **Espace suivi gamifié** (voir section GAM — identique à ELV-06, lecture seule pour le parent).   • Historique des cours : liste avec date, prof, matière, durée.   • Suivi post-séance du prof : pour chaque cours passé, les émojis et remarques du prof (hors notes privées).   • Suivi post-séance de l'élève : réponses au questionnaire élève.   • Devoirs : liste des devoirs assignés avec statut. **Critères d'acceptation** Parent voit les évaluations émoji du prof pour ses cours. Si le prof a coché 'besoin réunion' → un badge rouge 'Le prof souhaite vous rencontrer' s'affiche sur la fiche. Notes privées du prof → jamais visibles par le parent. Parent voit les badges et l'XP de son enfant. |  |

| PAR-05 | Section — Paiements |
| :---- | :---- |
| **Description** Le parent consulte son historique financier. **Comportement attendu** Tableau avec colonnes : Date, Cours (date \+ élève \+ prof), Montant payé, Statut (EN\_ATTENTE / REVERSE / REMBOURSE). Bouton 'Demander un remboursement' visible uniquement si la politique du prof l'autorise ET que le cours n'est pas passé ET dans les délais de la politique. Clic 'Demander un remboursement' → modale de confirmation avec rappel de la politique de remboursement → appel API qui déclenche un remboursement Stripe. **Critères d'acceptation** Remboursement éligible → bouton visible, clic → remboursement Stripe, statut passe à REMBOURSE. Remboursement non éligible → bouton absent. Statut EN\_ATTENTE → le cours est à venir ou \< 24h après la fin. Statut REVERSE → le prof a reçu son argent. |  |

| 🔵  7\. Dashboard Élève (mineur) *Accès aux cours, devoirs et ressources* |
| :---- |

IMPORTANT : Si l'élève n'a aucun cours à venir, l'intégralité du dashboard est remplacée par une page d'attente affichant : 'Aucun cours à venir pour le moment. Demandez à vos parents de réserver un cours \!' L'élève ne peut accéder à aucune autre page.

| ELV-01 | Page cours à venir |
| :---- | :---- |
| **Description** L'élève voit ses prochains cours. **Comportement attendu** Liste des cours à venir triée par date croissante. Pour chaque cours : date, heure, durée, prof (nom \+ photo), matière, lien Google Meet (bouton 'Rejoindre le cours' actif uniquement 15 minutes avant l'heure de début et jusqu'à 1h après). Statut du lien : 'Disponible dans X heures/minutes' ou 'Rejoindre maintenant' (bouton vert actif). **Critères d'acceptation** 15 min avant le cours → bouton 'Rejoindre maintenant' devient cliquable. Plus de 1h après la fin → lien désactivé. Aucun cours à venir → page de blocage affichée (voir intro section). |  |

| ELV-02 | Page cours passés |
| :---- | :---- |
| **Description** L'élève consulte l'historique de ses cours. **Comportement attendu** Liste des cours passés triée par date décroissante. Pour chaque cours : date, heure, prof, matière, lien vers la page devoirs de ce cours. Indicateur visuel si un devoir est associé à ce cours (badge 'Devoir' orange si non rendu, vert si rendu). |  |

| ELV-03 | Page devoir d'un cours |
| :---- | :---- |
| **Description** L'élève consulte et rend ses devoirs. **Comportement attendu** Titre du devoir, consigne complète (avec formatage), date limite de rendu, statut actuel. Pièces jointes du prof (boutons de téléchargement). Zone de rendu : textarea pour écrire sa réponse ET/OU upload de fichiers (max 3, 10 Mo chacun, vers R2). Bouton 'Rendre le devoir' → statut passe à RENDU, notification au prof. Si le devoir est CORRIGE : afficher la correction du prof dans un bloc distinct. **Critères d'acceptation** Rendu possible uniquement avant la date limite. Après la date limite → zone de rendu grisée, message 'Date limite dépassée'. Après correction → correction affichée dans un encadré vert. |  |

| ELV-04 | Page Mes ressources |
| :---- | :---- |
| **Description** L'élève accède aux ressources partagées par ses profs. **Comportement attendu** Liste des ressources : titre, type (fichier PDF/image/autre ou lien URL), prof qui l'a ajoutée, date d'ajout. Bouton 'Télécharger' ou 'Ouvrir le lien' selon le type. Filtre par prof (si plusieurs profs). **Critères d'acceptation** Si aucune ressource → message 'Votre professeur n'a pas encore partagé de ressources.' |  |

| ELV-05 | Questionnaire post-séance élève |
| :---- | :---- |
| **Description** Après chaque cours, l'élève est invité (mais pas obligé) à donner son ressenti. **Comportement attendu** Une bannière non-intrusive apparaît en haut du dashboard : '💬 Comment s'est passé ton cours de \[matière\] avec \[prof\] ?' Clic sur la bannière → ouvre un formulaire en modale avec 3 questions en émojis :   • Comment s'est passé le cours ? 😟 😐 🙂 😄 🤩   • Est-ce que tu as compris ? 😕 🤔 😐 👍 💡   • Comment tu te sens dans cette matière ? 😰 😟 😐 😊 💪 Bouton 'Envoyer' → sauvegarde en base. Bouton 'Passer' → bannière disparaît, questionnaire marqué comme ignoré. Le questionnaire n'est proposé qu'une fois par cours. **Critères d'acceptation** Les réponses sont visibles par le parent (section suivi) et le prof (fiche élève). Passer → aucune relance. |  |

| 🟣  8\. Dashboard Étudiant Autonome *Adulte : réserve, paie, suit ses cours lui-même* |
| :---- |

L'étudiant autonome est un adulte qui n'a pas de parent sur la plateforme. Son dashboard est un mixte entre le dashboard parent (réservation, paiement) et le dashboard élève (cours, devoirs, ressources). Les fonctionnalités sont strictement les mêmes, sauf qu'il n'y a pas de section 'Mes enfants'.

| Réservation de cours | Identique à PAR-03 — mais il réserve pour lui-même (pas de sélecteur d'élève) |
| :---- | :---- |
| **Paiements** | Identique à PAR-05 |
| **Cours à venir** | Identique à ELV-01 |
| **Cours passés** | Identique à ELV-02 |
| **Devoirs** | Identique à ELV-03 |
| **Mes ressources** | Identique à ELV-04 |
| **Questionnaire post-séance** | Identique à ELV-05 |
| **Suivi gamifié** | Identique à ELV-06 |
| **Ajouter un prof** | Identique à PAR-02 — via code d'invitation |
| **Pas de section enfants** | Non applicable |

| 🎮  8b. Système de suivi gamifié *Espace de progression bienveillant pour l'élève et le parent* |
| :---- |

> **Philosophie :** Le ton est toujours doux et encourageant, y compris en cas d'absence ou de devoir non rendu. On ne stigmatise jamais l'échec — on encourage le rebond et la régularité. L'élève gagne de l'XP pour ses actions positives ; les événements négatifs ne font pas perdre d'XP mais déclenchent un message de soutien.

| GAM-01 | Page Suivi — Vue élève (ELV-06) |
| :---- | :---- |
| **Description** Page de suivi gamifié accessible depuis le dashboard de l'élève (et en lecture seule depuis le dashboard du parent). **Comportement attendu — Bloc XP et niveau** : Barre de progression horizontale affichant l'XP totale de l'élève et le palier suivant. Ex. : '350 XP — Niveau Explorateur · Prochain niveau : Aventurier à 500 XP'. Les niveaux sont : Débutant (0–99 XP) → Curieux (100–249) → Explorateur (250–499) → Aventurier (500–899) → Expert (900–1499) → Maître (1500+ XP). **Comportement attendu — Graphiques (sur la même page)** : Graphique 1 — Courbe d'XP dans le temps (1 point par semaine, axe X = semaines, axe Y = XP cumulée). Graphique 2 — Radar ou barres des 3 scores post-séance moyens (Attitude, Compréhension, Confiance) calculés depuis les questionnaires du prof. Les deux graphiques sont sur la même page, l'un au-dessus de l'autre. Utiliser Recharts (bibliothèque React). **Comportement attendu — Badges** : Grille de badges obtenus (couleur) et à débloquer (grisés). Au survol d'un badge grisé : tooltip 'Complète X cours pour débloquer'. Badges définis dans la SPEC BDD (seed). Liste minimale à implémenter : 'Premier pas' (1er cours), 'Assidu' (5 cours), 'Régulier' (3 semaines de suite), 'Devoirs faits' (5 devoirs rendus), 'Super élève' (moyenne prof >= 4/5 sur 3 cours consécutifs), 'Streak ×5' (5 semaines consécutives), 'Maître' (1500 XP). **Comportement attendu — Streak** : Affichage du streak actuel avec une flamme animée. Ex. : '🔥 3 semaines consécutives !'. Si streak = 0, afficher 'Commence une nouvelle série cette semaine !' (jamais de message négatif). **Comportement attendu — Messages contextuels** : Après un cours présent → bannière verte 'Bravo ! Tu as terminé ton cours de [matière]. +10 XP'. Devoir rendu dans les délais → 'Super travail ! +15 XP'. Devoir non rendu → 'Pas de panique ! Il est encore temps de t'y mettre 💪' (jamais de formulation accusatoire). Évaluation prof basse (< 3/5) → 'Ton prof croit en toi. Quelques efforts supplémentaires et tu vas y arriver ! 🌟'. Nouveau badge débloqué → modale de félicitations avec animation. **Critères d'acceptation** La page se charge avec les données réelles de l'élève connecté. Les graphiques s'affichent même avec peu de données (1 seul cours). Si aucun cours encore → message d'accueil : 'Ton aventure commence bientôt ! Reserve ton premier cours pour gagner tes premiers XP 🚀'. Les badges grisés s'affichent toujours (catalogue complet visible). Les scores post-séance affichés sont uniquement les `remarquesPubliques` et les scores numériques — jamais les `notesPrivees` du prof. |  |

| GAM-02 | Calcul et attribution de l'XP |
| :---- | :---- |
| **Description** Règles métier définissant quand et combien d'XP est attribué. Ces règles s'exécutent côté serveur uniquement (jamais côté client). **Événements déclencheurs et XP attribuée** : Cours effectué (statut Booking TERMINE, élève présent selon PostSeanceProf.attendance si disponible) → +10 XP, création d'un XpEvent de type COURS_EFFECTUE. Devoir rendu avant la date limite → +15 XP, type DEVOIR_RENDU. Devoir rendu plus d'1 heure avant la date limite → +5 XP bonus supplémentaire, type DEVOIR_RENDU_EN_AVANCE. Questionnaire post-séance élève rempli → +5 XP, type QUESTIONNAIRE_REMPLI. Évaluation prof avec attitude >= 4 OU compréhension >= 4 → +10 XP, type EVALUATION_PROF_POSITIVE (une seule fois par cours). Streak : si l'élève a eu au moins 1 cours lors de chaque semaine des N dernières semaines → +20 XP au passage du pallier, type STREAK_SEMAINE (vérifier lors de chaque mise à jour du booking en TERMINE). **Comment déclencher** : L'attribution d'XP est faite dans des fonctions utilitaires server-side appelées : (1) depuis le cron qui passe les bookings en TERMINE (pour COURS_EFFECTUE et STREAK), (2) depuis l'API Route de rendu de devoir (pour DEVOIR_RENDU), (3) depuis l'API Route de soumission du questionnaire élève (pour QUESTIONNAIRE_REMPLI), (4) depuis l'API Route de soumission du questionnaire prof (pour EVALUATION_PROF_POSITIVE). Chaque attribution crée une ligne dans XpEvent ET met à jour EleveProgress.totalXp. **Vérification des badges** : Après chaque attribution d'XP, appeler une fonction `checkAndAwardBadges(eleveId)` qui vérifie les conditions de tous les badges non encore obtenus et crée les EleveBadge manquants. Si un nouveau badge est créé, mettre à jour `EleveProgress.pendingBadgeSlug` avec le `slug` du badge obtenu. À la prochaine page vue de l'élève, si `pendingBadgeSlug` est non-null : afficher la modale de félicitations, puis remettre `pendingBadgeSlug` à null. **Critères d'acceptation** Un même cours ne peut déclencher COURS_EFFECTUE qu'une seule fois. Un même devoir ne peut déclencher DEVOIR_RENDU qu'une seule fois. EleveProgress.totalXp est toujours égal à la somme de tous les XpEvent.xpGained de l'élève. |  |

| 💳  9\. Flux de paiement Stripe *Encaissement, escrow, reversement, remboursement* |
| :---- |

| PAY-01 | Encaissement à la réservation |
| :---- | :---- |
| **Description** Lorsque le parent/étudiant valide sa réservation, le paiement est encaissé immédiatement. **Comportement attendu** API Route POST /api/bookings/create reçoit : profId, eleveId, creneaux\[\] (liste de dates/heures), payerId. Vérification que chaque créneau est toujours disponible (pas de double-réservation). Calcul du montant total : tarif\_horaire × nombre\_de\_créneaux × durée. Création d'une session Stripe Checkout avec :   • payment\_intent\_data.capture\_method \= 'automatic'   • application\_fee\_amount \= montant × PLATFORM\_FEE\_PERCENT / 100 (en centimes)   • transfer\_data.destination \= stripeAccountId du prof   • metadata : bookingIds\[\], profId, payerId Redirection vers la page Stripe Checkout hébergée par Stripe. Stripe redirige vers /booking/success?session\_id=xxx ou /booking/cancel. **Critères d'acceptation** Paiement réussi → webhook Stripe checkout.session.completed reçu → création des Booking en base avec statut CONFIRME, création de l'événement Google Calendar avec lien Meet. Paiement annulé → redirection /booking/cancel, aucun booking créé. Créneau pris entre la sélection et le paiement → le webhook détecte la collision, remboursement automatique Stripe, email d'explication au parent. |  |

| PAY-02 | Marquage applicatif du reversement (pas de transfert Stripe) |
| :---- | :---- |
| **Description** En mode *destination charges* (POC), Stripe a déjà routé l'argent vers le compte connecté du prof au moment du Checkout. Il n'y a donc PAS de transfert à déclencher. Ce cron sert uniquement à **basculer le statut applicatif** de `EN_ATTENTE` à `REVERSE` une fois le cours passé depuis plus de 24h, et à notifier le prof. **Comportement attendu** Vercel Cron Job toutes les heures, appelle GET /api/cron/payouts (header `Authorization: Bearer <CRON_SECRET>` obligatoire). Cible : `Payment` avec `status = EN_ATTENTE` ET `booking.endTime < now() - 24h`. Action : `Payment.status = REVERSE`, `payoutSentAt = now()`. Email EMAIL-04 au prof. **Critères d'acceptation** Aucune API Stripe Transfer n'est appelée. Idempotence garantie par la condition `status = EN_ATTENTE`. Cron sans `CRON_SECRET` → 401. |  |

| PAY-03 | Remboursement |
| :---- | :---- |
| **Description** Le parent peut demander un remboursement selon la politique du prof. **Comportement attendu** API Route POST /api/bookings/\[bookingId\]/refund. Vérifications avant remboursement :   • L'utilisateur qui demande est bien le payeur du booking.   • Le cours n'est pas encore passé.   • La politique de remboursement du prof autorise le remboursement (délai respecté).   • Le statut du booking est CONFIRME (pas déjà REVERSE ou REMBOURSE). Si toutes les conditions OK : appel Stripe Refund sur le PaymentIntent. Mise à jour en base : statut Booking \= ANNULE, statut Payment \= REMBOURSE. Suppression de l'événement Google Calendar associé. Emails : confirmation de remboursement au parent \+ notification d'annulation au prof. **Critères d'acceptation** Remboursement réussi → statut REMBOURSE en base, email parent et prof. Politique non respectée → 403 avec message explicite. Cours déjà passé → 403 'Ce cours est déjà passé, impossible de rembourser.' |  |

| 📅  10\. Google Calendar & Google Meet *Synchronisation des disponibilités et création des cours* |
| :---- |

| GCal-01 | Lecture des disponibilités |
| :---- | :---- |
| **Description** Calculer les créneaux libres du prof. **Source de vérité : la table `ProfAvailability`** (plages hebdomadaires récurrentes en base). Google Calendar n'est PAS interrogé pour cette lecture — choix POC pour éviter quota, latence et fragilité d'une convention de titre. **Comportement attendu** API Route GET /api/prof/\[profId\]/slots?from=DATE\&to=DATE. Étapes : (1) charger les `ProfAvailability` du prof, (2) projeter ces plages sur la fenêtre demandée en respectant le fuseau du prof, (3) exclure les créneaux déjà occupés par un `Booking` (status `CONFIRME` ou `TERMINE`) en base. Retourner un tableau de créneaux libres avec : startTime, endTime. **Critères d'acceptation** Réponse purement BDD, pas d'appel Google. Créneaux dans le passé jamais retournés. Créneau déjà réservé jamais retourné. |  |

| GCal-02 | Création d'un événement cours avec Google Meet |
| :---- | :---- |
| **Description** À la confirmation du paiement, créer l'événement dans Google Calendar avec un lien Meet. **Comportement attendu** Déclenché par le webhook Stripe checkout.session.completed. Pour chaque booking confirmé : appel Google Calendar API events.insert avec :   • summary : '\[COURS\] Matière avec Élève'   • start et end : date/heure du créneau   • attendees : email du prof \+ email de l'élève (ou parent si l'élève n'a pas d'email)   • conferenceData.createRequest : pour générer le lien Google Meet automatiquement   • conferenceDataVersion \= 1 dans la requête Stocker le lien Meet (hangoutLink) dans le champ meetLink du Booking en base. Le lien Meet est ensuite affiché dans les dashboards prof, parent et élève. **Critères d'acceptation** Création réussie → meetLink stocké en base. Création échouée → booking quand même confirmé, meetLink \= null, afficher 'Lien de visio indisponible — contactez le prof'. |  |

| 📧  11\. Emails transactionnels (Resend) *Liste exhaustive de tous les emails envoyés par la plateforme* |
| :---- |

| EMAIL-01 | Confirmation de réservation → Parent/Étudiant. Déclenché par : webhook Stripe checkout.session.completed. Contenu : résumé des cours réservés, dates, profs, montant total payé, politique de remboursement. |
| :---- | :---- |
| **EMAIL-02** | Confirmation de réservation → Prof. Déclenché par : webhook Stripe checkout.session.completed. Contenu : nouveau cours réservé, date/heure, élève, montant qu'il recevra. |
| **EMAIL-03** | Rappel de cours J-1 → Parent/Étudiant \+ Prof \+ Élève. Déclenché par : Vercel Cron Job quotidien à 08h00. Contenu : rappel date/heure, lien Google Meet, nom du prof/élève. |
| **EMAIL-04** | Confirmation de reversement → Prof. Déclenché par : cron PAY-02. Contenu : montant reversé, date du cours concerné. |
| **EMAIL-05** | Confirmation de remboursement → Parent/Étudiant. Déclenché par : PAY-03. Contenu : montant remboursé, délai bancaire estimé (3-5 jours ouvrés). |
| **EMAIL-06** | Notification annulation → Prof. Déclenché par : PAY-03. Contenu : cours annulé, date/heure. |
| **EMAIL-07** | Invitation élève → Élève. Déclenché par : PAR-04 (parent ajoute un enfant avec email). Contenu : lien de création de compte avec code de liaison pré-rempli. |
| **EMAIL-08** | Demande de réunion parent → Parent. Déclenché par : PROF-07 (prof coche 'besoin réunion \= Oui'). Contenu : 'Le professeur \[nom\] souhaite vous rencontrer pour parler des progrès de \[enfant\].' |
| **EMAIL-09** | Devoir rendu → Prof. Déclenché par : ELV-03 (élève rend un devoir). Contenu : nom élève, titre devoir, lien vers le rendu. |

| 🗄️  12\. Schéma de base de données (Prisma / PostgreSQL 16) *Entités, champs et relations* |
| :---- |

> Le schéma complet et détaillé avec les relations Prisma est dans **SPEC BDD.md**. Cette section est un résumé rapide à destination de l'agent de développement.

### **12.1 Entité User**

| id | String — UUID — clé primaire |
| :---- | :---- |
| **email** | String — unique — obligatoire |
| **emailVerified** | DateTime? — nullable |
| **username** | String — **unique** — pseudo public (pas de nom/prénom obligatoire — RGPD) |
| **image** | String? — URL photo (R2) |
| **passwordHash** | String? — nullable si OAuth uniquement |
| **role** | Enum : ADMIN / PROF / PARENT / ELEVE / ETUDIANT |
| **bannedAt** | DateTime? — null si actif |
| **cguAcceptedAt** | DateTime? — horodatage acceptation CGU (RGPD) |
| **createdAt** | DateTime — défaut : now() |
| **updatedAt** | DateTime — auto-updated |
| **onboardingCompleted** | Boolean — défaut : false — pour les PROF uniquement |

### **12.2 Entité ProfProfile**

| id | String — UUID |
| :---- | :---- |
| **userId** | String — FK → User (relation 1-1) |
| **bio** | String? — max 500 chars |
| **hourlyRate** | Int — en centimes |
| **subjects** | String\[\] — tableau de matières |
| **levels** | String\[\] — tableau de niveaux |
| **stripeAccountId** | String? — Stripe Connect Express ID |
| **googleCalendarEmail** | String? — email Google connecté |
| **refundPolicy** | Enum : REFUND\_2H / REFUND\_24H / REFUND\_48H / NO\_REFUND |
| **refundPolicyNote** | String? — précisions libres |
| **inviteCode** | String? — code d'invitation actif |
| **inviteCodeExpiresAt** | DateTime? |

### **12.3 Entité Account (Auth.js — OAuth tokens)**

| id | String — UUID |
| :---- | :---- |
| **userId** | String — FK → User |
| **type** | String — 'oauth' |
| **provider** | String — 'google' ou 'credentials' |
| **providerAccountId** | String |
| **access\_token** | String? — chiffré |
| **refresh\_token** | String? — chiffré |
| **expires\_at** | Int? |
| **scope** | String? |

### **12.4 Entité SessionLog**

| id | String — UUID |
| :---- | :---- |
| **userId** | String — FK → User |
| ~~ipAddress~~ | **Supprimé — non collecté (RGPD)** |
| **userAgent** | String? — navigateur uniquement |
| **createdAt** | DateTime — défaut : now() |

### **12.5 Entité ProfParent (relation prof ↔ parent)**

| id | String — UUID |
| :---- | :---- |
| **profId** | String — FK → User (rôle PROF) |
| **parentId** | String — FK → User (rôle PARENT ou ETUDIANT) |
| **linkedAt** | DateTime — défaut : now() |

### **12.6 Entité ParentEleve (relation parent ↔ élève)**

| id | String — UUID |
| :---- | :---- |
| **parentId** | String — FK → User (rôle PARENT) |
| **eleveId** | String — FK → User (rôle ELEVE) |
| **parentalConsentAt** | DateTime — consentement parental RGPD-Kids à la création |
| **linkedAt** | DateTime — défaut : now() |

### **12.7 Entité Booking**

| id | String — UUID |
| :---- | :---- |
| **profId** | String — FK → User (PROF) |
| **eleveId** | String — FK → User (ELEVE ou ETUDIANT) |
| **payerId** | String — FK → User (PARENT ou ETUDIANT) |
| **startTime** | DateTime — heure de début (UTC) |
| **endTime** | DateTime — heure de fin (UTC) |
| **status** | Enum : CONFIRME / ANNULE / TERMINE |
| **meetLink** | String? — lien Google Meet |
| **googleEventId** | String? — ID événement Google Calendar |
| **createdAt** | DateTime |

### **12.8 Entité Payment**

| id | String — UUID |
| :---- | :---- |
| **bookingId** | String — FK → Booking (1-1) |
| **stripePaymentIntentId** | String — unique |
| **stripeCheckoutSessionId** | String — unique |
| **amount** | Int — montant total en centimes |
| **platformFee** | Int — commission en centimes |
| **profAmount** | Int — montant reversé au prof en centimes |
| **status** | Enum : EN\_ATTENTE / REVERSE / REMBOURSE |
| **payoutSentAt** | DateTime? — null si pas encore reversé |
| **refundedAt** | DateTime? — null si pas remboursé |
| **createdAt** | DateTime |

### **12.9 Entité PostSeanceProf**

| id | String — UUID |
| :---- | :---- |
| **bookingId** | String — FK → Booking (1-1) |
| **attendance** | Boolean — true = présent, false = absent |
| **attitude** | Int — 1 à 5 (icône Lucide) |
| **comprehension** | Int — 1 à 5 |
| **confiance** | Int — 1 à 5 |
| **needsMeeting** | Boolean |
| **remarquesPubliques** | String? — max 1000 chars — visible parent |
| **notesPrivees** | String? — max 1000 chars — visible prof uniquement |
| **createdAt** | DateTime |

### **12.10 Entité PostSeanceEleve**

| id | String — UUID |
| :---- | :---- |
| **bookingId** | String — FK → Booking (1-1) |
| **ressenti** | Int — 1 à 5 |
| **comprehension** | Int — 1 à 5 |
| **confianceMatiere** | Int — 1 à 5 |
| **createdAt** | DateTime |

### **12.11 Entité Homework**

| id | String — UUID |
| :---- | :---- |
| **profId** | String — FK → User (PROF) |
| **eleveId** | String — FK → User (ELEVE ou ETUDIANT) |
| **bookingId** | String? — FK → Booking (nullable, devoir pas forcément lié à 1 cours) |
| **title** | String |
| **description** | String — contenu riche (Markdown ou HTML sanitisé) |
| **dueDate** | DateTime |
| **status** | Enum : NON\_RENDU / RENDU / CORRIGE |
| **renderedContent** | String? — réponse de l'élève (texte) |
| **correctionContent** | String? — correction du prof |
| **createdAt** | DateTime |

### **12.12 Entité Resource**

| id | String — UUID |
| :---- | :---- |
| **profId** | String — FK → User (PROF) |
| **eleveId** | String? — FK → User — null \= visible par tous les élèves du prof |
| **title** | String |
| **type** | Enum : FICHIER / LIEN |
| **url** | String — URL R2 ou URL externe |
| **createdAt** | DateTime |

### **12.13 Entité HomeworkFile**

Fichiers attachés à un devoir (max 3, 10 Mo chacun, Cloudflare R2).

| id | String — UUID |
| :---- | :---- |
| **homeworkId** | String — FK → Homework |
| **url** | String — URL R2 |
| **filename** | String — nom original du fichier |
| **size** | Int — taille en bytes |
| **createdAt** | DateTime |

### **12.13b Entité ResourceFile**

Fichiers attachés à une ressource de type FICHIER (max 3, 10 Mo chacun, Cloudflare R2).

| id | String — UUID |
| :---- | :---- |
| **resourceId** | String — FK → Resource |
| **url** | String — URL R2 |
| **filename** | String — nom original du fichier |
| **size** | Int — taille en bytes |
| **createdAt** | DateTime |

### **12.14 Entité EleveProgress** ← NOUVEAU

| id | String — UUID |
| :---- | :---- |
| **eleveId** | String — unique — FK → User |
| **totalXp** | Int — défaut 0 — XP totale cumulée |
| **currentStreak** | Int — défaut 0 — semaines consécutives en cours |
| **longestStreak** | Int — défaut 0 — record personnel |
| **totalCours** | Int — défaut 0 — nombre de cours effectués |
| **pendingBadgeSlug** | String? — slug du badge à notifier à la prochaine page vue, null si aucun |
| **updatedAt** | DateTime — auto-updated |

### **12.15 Entité XpEvent** ← NOUVEAU

| id | String — UUID |
| :---- | :---- |
| **eleveId** | String — FK → User |
| **type** | Enum : COURS_EFFECTUE / DEVOIR_RENDU / DEVOIR_RENDU_EN_AVANCE / EVALUATION_PROF_POSITIVE / QUESTIONNAIRE_REMPLI / STREAK_SEMAINE |
| **xpGained** | Int — points gagnés |
| **bookingId** | String? — FK → Booking |
| **homeworkId** | String? — FK → Homework |
| **label** | String? — texte affiché ex. 'Cours de Maths ✓ +10 XP' |
| **createdAt** | DateTime |

### **12.16 Entité Badge** ← NOUVEAU

| id | String — UUID |
| :---- | :---- |
| **slug** | String — unique — identifiant technique ex. 'premier-cours' |
| **label** | String — nom affiché ex. 'Premier pas !' |
| **description** | String — condition lisible ex. 'Tu as terminé ton premier cours' |
| **icon** | String — nom d'icône Lucide UNIQUEMENT (ex. 'Star', 'Flame', 'Trophy') — **pas d'emoji** |
| **xpRequired** | Int? — seuil XP si condition = XP |

### **12.17 Entité EleveBadge** ← NOUVEAU

| id | String — UUID |
| :---- | :---- |
| **eleveId** | String — FK → User |
| **badgeId** | String — FK → Badge |
| **obtainedAt** | DateTime |
| **Contrainte** | UNIQUE(eleveId, badgeId) — un badge ne peut être obtenu qu'une seule fois |

| ⚙️  13\. Variables d'environnement requises *Toutes les variables à configurer avant de démarrer* |
| :---- |

| DATABASE\_URL | URL de connexion PostgreSQL Neon |
| :---- | :---- |
| **NEXTAUTH\_URL** | URL publique du site (ex: https://monsite.com) |
| **NEXTAUTH\_SECRET** | Secret aléatoire pour Auth.js (générer avec openssl rand \-base64 32\) |
| **GOOGLE\_CLIENT\_ID** | Client ID OAuth Google (Google Cloud Console) |
| **GOOGLE\_CLIENT\_SECRET** | Client Secret OAuth Google |
| **STRIPE\_SECRET\_KEY** | Clé secrète Stripe (sk\_live\_... ou sk\_test\_...) |
| **STRIPE\_PUBLISHABLE\_KEY** | Clé publique Stripe (pk\_live\_... ou pk\_test\_...) |
| **STRIPE\_WEBHOOK\_SECRET** | Secret de signature webhook Stripe (whsec\_...) |
| **STRIPE\_CONNECT\_CLIENT\_ID** | Client ID pour Stripe Connect (ca\_...) |
| **PLATFORM\_FEE\_PERCENT** | Pourcentage de commission (défaut : 10\) |
| **RESEND\_API\_KEY** | Clé API Resend (re\_...) |
| **RESEND\_FROM\_EMAIL** | Email expéditeur (ex: noreply@monsite.com) |
| **R2\_ACCOUNT\_ID** | ID compte Cloudflare |
| **R2\_ACCESS\_KEY\_ID** | Access Key Cloudflare R2 |
| **R2\_SECRET\_ACCESS\_KEY** | Secret Key Cloudflare R2 |
| **R2\_BUCKET\_NAME** | Nom du bucket R2 |
| **R2\_PUBLIC\_URL** | URL publique du bucket R2 |
| **CRON\_SECRET** | Secret header pour protéger les routes /api/cron/\* |

| ✅  14\. Checklist de validation (agent IA) *Cocher chaque item avant de passer à la feature suivante* |
| :---- |

| L'agent doit cocher chaque critère avant de marquer une feature comme DONE. Si un critère échoue, la feature est en statut BLOCKED et doit être corrigée avant de continuer. |
| :---- |

### **Pages publiques**

* \[ \] PUB-01 : L'utilisateur connecté sur / est redirigé vers son dashboard

* \[ \] PUB-01 : L'utilisateur non connecté voit le bouton 'Se connecter'

* \[ \] PUB-02 : Mauvais mot de passe → message générique sans préciser email ou mdp

* \[ \] PUB-02 : OAuth Google → email inconnu → message d'erreur, pas de création auto

* \[ \] PUB-03 : Email doublon → message d'erreur, pas de création

* \[ \] PUB-03 : Rôle PROF → redirection onboarding après inscription

### **Onboarding prof**

* \[ \] ONB-01 : Tarif hors plage → erreur

* \[ \] ONB-02 : Google Calendar non connecté → bouton Suivant désactivé

* \[ \] ONB-03 : Stripe non connecté → bouton Suivant désactivé

* \[ \] ONB-04 : Onboarding terminé → accès au dashboard prof débloqué

* \[ \] Toute route /dashboard/prof/\* sans onboarding terminé → redirection /onboarding

### **Dashboard prof**

* \[ \] PROF-03 : Sauvegarde disponibilités → événements créés dans Google Calendar

* \[ \] PROF-06 : Devoir créé → visible dans dashboard élève ciblé

* \[ \] PROF-07 : Questionnaire apparaît 15 min après fin de cours

* \[ \] PROF-07 : needsMeeting \= true → email envoyé au parent

* \[ \] PROF-08 : Code d'invitation expiré → message d'erreur correct

### **Dashboard parent**

* \[ \] PAR-03 : Créneau déjà réservé → grisé, non cliquable

* \[ \] PAR-03 : Aucun créneau sélectionné → bouton paiement désactivé

* \[ \] PAR-04 : Notes privées du prof → invisibles pour le parent

* \[ \] PAR-05 : Remboursement hors délai → bouton absent

### **Dashboard élève**

* \[ \] ELV : Aucun cours à venir → page de blocage, aucune autre page accessible

* \[ \] ELV-01 : Lien Meet cliquable uniquement 15 min avant et jusqu'à 1h après

* \[ \] ELV-03 : Rendu après date limite → zone grisée

* \[ \] ELV-05 : Questionnaire proposé une seule fois par cours

* \[ \] ELV-06 : Page suivi gamifié accessible depuis le dashboard élève

* \[ \] GAM-01 : Les deux graphiques (XP dans le temps + radar scores) s'affichent sur la même page

* \[ \] GAM-01 : Badge débloqué → modale de félicitations affichée à la prochaine page vue

* \[ \] GAM-01 : Aucun message négatif — ton toujours bienveillant

* \[ \] GAM-02 : Un même cours ne déclenche COURS_EFFECTUE qu'une seule fois

* \[ \] GAM-02 : EleveProgress.totalXp == somme de tous les XpEvent de l'élève

* \[ \] GAM-02 : Parent voit le suivi gamifié de ses enfants en lecture seule

### **Paiements**

* \[ \] PAY-01 : Double-réservation détectée → remboursement automatique \+ email

* \[ \] PAY-01 : Montants manipulés en centimes uniquement (pas de float)

* \[ \] PAY-02 : Cron tourne toutes les heures sans traiter 2x le même booking

* \[ \] PAY-03 : Politique non respectée → 403 avec message

### **Google Calendar**

* \[ \] GCal-01 : Token expiré → refresh automatique

* \[ \] GCal-02 : Événement créé avec lien Meet après paiement

* \[ \] GCal-02 : Si création échoue → booking quand même confirmé, meetLink \= null

### **Emails**

* \[ \] EMAIL-01/02 envoyés après confirmation paiement

* \[ \] EMAIL-03 envoyé J-1 à 08h00

* \[ \] EMAIL-04 envoyé après chaque reversement

* \[ \] EMAIL-05/06 envoyés après remboursement

### **Sécurité**

* \[ \] Chaque API Route vérifie le rôle avant toute action

* \[ \] Webhook Stripe vérifié par signature (stripe.webhooks.constructEvent)

* \[ \] Route /api/cron/\* protégée par CRON\_SECRET header

* \[ \] Aucun float dans les calculs de montant

* \[ \] Notes privées du prof jamais renvoyées dans les requêtes parent/élève

| 🚫  15\. Hors scope v1 — Ne pas implémenter *Features explicitement exclues de cette version* |
| :---- |

| L'agent NE DOIT PAS implémenter ces features même si elles semblent logiques. Elles sont réservées à une version future. |
| :---- |

* Moteur de recherche de professeurs (mise en relation)

* Chat ou messagerie interne entre prof et parent/élève

* Visioconférence intégrée custom (on utilise Google Meet via Calendar API)

* Application mobile native (iOS / Android)

* Génération de factures PDF

* Système de notation/avis des professeurs

* Abonnements ou packs de cours prépayés

* Paiement en plusieurs fois

* Tableau de bord analytique avancé pour l'admin ou le prof (KPI business — **le suivi élève avec graphiques XP/scores est, lui, bien dans le scope**)

* Système de parrainage

* Multi-langue (v1 \= français uniquement)

* Import/export CSV des données

* Notifications push (web push / app mobile)