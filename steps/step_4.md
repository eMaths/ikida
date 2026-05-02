# STEP 4 — Liaison prof-parent, disponibilités et calendrier de réservation

**Objectif :** un parent/étudiant rejoint un prof via un code d'invitation. Le prof gère ses disponibilités via Google Calendar. Le parent peut voir les créneaux disponibles et les sélectionner (sans payer encore — le paiement est step_5).
**Dépend de :** step_2 et step_3.
**Branche :** `feat/booking-calendar`

---

## Ce que tu livres à la fin de cette étape

- Système de codes d'invitation (génération, expiration, utilisation)
- Client Google Calendar centralisé dans `/lib/google-calendar.ts`
- API de gestion des disponibilités prof
- Page de réservation avec calendrier de sélection de créneaux
- Dashboard prof (KPIs) et dashboard parent (liste profs, bouton réserver)
- Gestion des comptes enfants par le parent

---

## Ordre d'exécution

### 1. Client Google Calendar

Crée `/lib/google-calendar.ts`.

Ce fichier est le **seul endroit** où le Google Calendar API est appelé dans tout le projet.
Responsabilités :
- Récupérer le `refresh_token` du prof depuis la table `Account`
- Créer un client OAuth2 Google avec les credentials depuis `process.env`
- Rafraîchir automatiquement le token si expiré
- Exposer des fonctions : `getAvailableSlots(profId, startDate, endDate)`, `createCourseEvent(profId, eventData)`, `deleteCourseEvent(profId, eventId)`

`getAvailableSlots` : lit les événements du calendrier Google du prof et retourne les créneaux libres selon ses disponibilités sauvegardées en base. Ne jamais exposer les événements personnels du prof — uniquement calculer les créneaux libres à partir des plages de disponibilité.

`createCourseEvent` : crée un événement avec Google Meet (`conferenceData`), retourne `{ googleEventId, meetLink }`.

### 2. Code d'invitation

Crée `POST /api/prof/invite`.

Génère un code UUID court (8 chars), l'enregistre sur `ProfProfile.inviteCode` avec une expiration à +7 jours. Si un code actif existe déjà, le renouveler.
Retourne `{ code, expiresAt }`.

Crée `POST /api/join/[code]`.

Ordre des opérations :
1. Session obligatoire (PARENT ou ETUDIANT)
2. Retrouver le prof par `inviteCode`
3. Vérifier `inviteCodeExpiresAt` — code expiré → 410
4. Vérifier qu'un lien `ProfParent` n'existe pas déjà
5. Créer le `ProfParent`
6. Retourner `{ profUsername, profId }`

Crée `/join/[code]` côté public :
- Si pas de session → afficher les infos du prof + boutons "Se connecter" / "Créer un compte"
- Si session PARENT/ETUDIANT → appelle l'API et redirige vers le dashboard

### 3. Disponibilités prof

Crée `POST /api/prof/availability`.

Reçoit un tableau de plages hebdomadaires : `{ dayOfWeek: 0-6, startHour: number, endHour: number }[]`.
Sauvegarde dans `ProfAvailability` (remplace toutes les disponibilités existantes du prof en transaction).

Crée `GET /api/prof/[profId]/slots`.

Paramètres : `startDate`, `endDate`.
Calcule les créneaux libres à partir des disponibilités sauvegardées, croise avec les bookings existants pour exclure les créneaux déjà pris.
Retourne une liste de créneaux `{ startTime, endTime }`.
Ne retourne que les créneaux dans le futur.

### 4. Comptes enfants

Crée `POST /api/parent/children`.

Ordre :
1. Session PARENT
2. Validation Zod : `username`, `birthYear`
3. Créer un `User` avec rôle `ELEVE`, email interne `{username}@ikida.internal`, pas de mot de passe (`passwordHash = null`)
4. Créer `EleveProgress` lié
5. Créer `ParentEleve` pour lier parent et enfant — `parentalConsentAt = now()` (l'acte de création par le parent vaut consentement RGPD-Kids)
6. Envoyer EMAIL-07 au parent (Resend) avec le username du compte créé

Crée `GET /api/parent/children` : liste les enfants du parent connecté.

### 5. Dashboards

**Dashboard prof** `/dashboard/prof` :
- Prochain cours (next booking CONFIRME)
- Nombre d'élèves actifs
- Total encaissé ce mois (en euros, calculé depuis `Payment.profAmount`)
- Questionnaires en attente (bookings TERMINE sans `PostSeanceProf`)
- Bouton "Générer lien d'invitation"

**Dashboard parent** `/dashboard/parent` :
- Liste des profs liés avec bouton "Réserver"
- Liste des enfants avec accès à leur fiche

**Calendrier de réservation** `/booking/[profId]` :
- Accessible aux PARENT et ETUDIANT
- Affiche les créneaux disponibles du prof mois par mois
- Sélection de plusieurs créneaux
- Sélecteur d'élève (pour les PARENT)
- Sélecteur de matière
- Bouton "Procéder au paiement" → appelle step_5

Le bouton de paiement est présent mais affiche "Fonctionnalité bientôt disponible" jusqu'à step_5.

### 6. Sidebar et navigation

Crée les sidebars prof et parent avec la navigation complète (tous les liens, même ceux qui pointent vers des pages à venir).
Une sidebar = un composant client avec `usePathname` pour l'état actif.

---

## KPI de validation

- Prof génère un code → `SELECT "inviteCode", "inviteCodeExpiresAt" FROM "ProfProfile"` → valeurs correctes
- Code expiré → `/api/join/[code]` → 410
- Parent rejoint avec un code valide → `SELECT * FROM "ProfParent"` → 1 ligne
- Double liaison → 409
- Prof sauvegarde ses disponibilités → `SELECT * FROM "ProfAvailability"` → lignes créées
- `GET /api/prof/[profId]/slots` retourne uniquement des créneaux futurs
- Un créneau déjà booké n'apparaît pas dans les slots
- Créer un enfant → `SELECT email FROM "User" WHERE role = 'ELEVE'` → se termine par `@ikida.internal`
- `tsc --noEmit` et `eslint` → 0 erreur

---

## Rappels architecturaux

- Toute la logique Google Calendar passe par `/lib/google-calendar.ts` — jamais d'appel direct à l'API Google dans une route ou un composant
- Les tokens OAuth (`access_token`, `refresh_token`) ne sont jamais loggués
- Les créneaux disponibles sont calculés côté serveur — jamais côté client
- Le `ProfParent` utilise `parentId` même pour un ETUDIANT (l'étudiant est son propre "parent" dans la relation)
