# STEP 7 — Questionnaires post-séance

**Objectif :** après chaque cours terminé, le prof évalue l'élève (présence, 3 scores, remarques). L'élève donne son ressenti. Le parent voit les scores publics. Les `notesPrivees` ne quittent jamais le périmètre prof.
**Dépend de :** step_6.
**Branche :** `feat/post-seance`

---

## Ce que tu livres à la fin de cette étape

- Questionnaire prof (PROF-07) : formulaire dans une modale, déclenché par une bannière
- Questionnaire élève (ELV-05) : formulaire optionnel, une seule fois par cours
- Page suivi enfant côté parent : scores publics uniquement
- `notesPrivees` : jamais exposé hors du périmètre PROF

---

## Ordre d'exécution

### 1. Questionnaire prof

Crée `POST /api/prof/post-seance`.

Ordre des opérations :
1. Session PROF
2. Validation Zod : `bookingId`, `attendance` (boolean, obligatoire), `attitude` (1-5), `comprehension` (1-5), `confiance` (1-5), `needsMeeting` (boolean), `remarquesPubliques` (max 1000), `notesPrivees` (max 1000)
3. Vérifier `booking.profId === session.user.id`
4. Vérifier `booking.status === 'TERMINE'`
5. `prisma.postSeanceProf.create` — l'unique contrainte sur `bookingId` protège l'idempotence
6. Si `needsMeeting = true` → envoyer EMAIL-08 au parent de l'élève
7. En cas de contrainte unique violée (P2002) → 409 "Questionnaire déjà rempli"

**Côté UI :**
Le dashboard prof (`/dashboard/prof`) charge les bookings `TERMINE` sans `PostSeanceProf` associé.
Pour chacun → afficher une bannière cliquable.
Clic → ouvre une modale `Dialog` (shadcn/ui) avec le formulaire.

Formulaire :
- Présence : deux boutons toggle Présent/Absent — **obligatoire avant tout le reste**
- Si Absent → les 3 scores sont masqués et envoyés à 0
- Scores 1 à 5 : représentés par des icônes Lucide (pas d'emoji, pas d'étoiles HTML — uniquement `lucide-react`)
- `needsMeeting` : toggle Oui/Non avec avertissement "Un email sera envoyé au parent"
- `remarquesPubliques` : textarea, visible par l'élève et le parent
- `notesPrivees` : textarea, labellisé explicitement "Visibles uniquement par vous"
- Bouton submit désactivé tant que `attendance` et `needsMeeting` ne sont pas renseignés

### 2. Questionnaire élève

Crée `POST /api/eleve/post-seance`.

Validations :
- `booking.eleveId === session.user.id`
- `booking.status === 'TERMINE'`
- Contrainte unique sur `bookingId` → 409 si déjà rempli

Reçoit : `bookingId`, `ressenti` (1-5), `comprehension` (1-5), `confianceMatiere` (1-5). Pas de champ commentaire (cf. SPEC BDD `PostSeanceEleve`).

Après succès : appeler `addXpEvent(eleveId, 'QUESTIONNAIRE_REMPLI', bookingId)` et `checkAndAwardBadges(eleveId)` (TODO si step_8 pas encore fait).

**Côté UI :**
Le dashboard élève charge les bookings `TERMINE` sans `PostSeanceEleve`.
Bannière discrète pour chaque cours en attente.
Bouton "Passer" → dismiss local (sessionStorage), pas de relance.
Bouton "Donner mon avis" → modale avec le formulaire.
Icônes Lucide uniquement pour les scores.

### 3. Suivi enfant côté parent

Crée `GET /api/parent/children/[childId]/progress`.

Vérifications :
1. Session PARENT
2. `ParentEleve` lié : `parentId === session.user.id && eleveId === childId`

Retourne les bookings `TERMINE` avec leur `PostSeanceProf` — `select` explicite :
- Inclure : `attendance`, `attitude`, `comprehension`, `confiance`, `needsMeeting`, `remarquesPubliques`, `createdAt`
- **Ne jamais inclure `notesPrivees`** dans ce `select` — c'est la règle la plus importante de cette étape

Page `/dashboard/parent/children/[childId]` :
- Historique des cours avec scores sous forme visuelle
- `remarquesPubliques` affiché
- `notesPrivees` : absent, même pas mentionné dans l'UI

### 4. Audit de sécurité post-implémentation

Après avoir tout codé, faire une recherche globale :

`grep -rn "notesPrivees" ./app ./lib ./components`

Résultat attendu : uniquement dans `/api/prof/post-seance` (écriture) et dans le composant du formulaire prof (saisie).
**Toute occurrence ailleurs est un bug de sécurité à corriger immédiatement.**

---

## KPI de validation

- Booking `TERMINE` sans questionnaire → bannière affichée dans le dashboard prof
- Remplir le questionnaire → `PostSeanceProf` créé, bannière disparaît
- Remplir une 2e fois → 409
- `needsMeeting = true` → EMAIL-08 reçu par le parent
- `attendance = false` → scores à 0 en base, pas de champs scores affichés dans l'UI
- Dashboard parent → page suivi enfant → `notesPrivees` absent du JSON de réponse (vérifier DevTools Network)
- `grep -rn "notesPrivees" ./app ./lib ./components` → uniquement dans le périmètre PROF
- Questionnaire élève → "Passer" → pas de relance après rechargement
- `tsc --noEmit` et `eslint` → 0 erreur

---

## Rappels architecturaux

- `notesPrivees` est la donnée la plus sensible de la plateforme — vérification systématique de chaque `select` Prisma qui touche à `PostSeanceProf`
- L'idempotence des questionnaires est garantie par la contrainte `@unique` en base, pas par du code applicatif
- Les bannières de questionnaires sont des données chargées côté serveur dans le layout ou la page — pas de polling côté client
