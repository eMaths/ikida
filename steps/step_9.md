# STEP 9 — Dashboard étudiant autonome

**Objectif :** l'étudiant autonome (ETUDIANT) peut réserver et payer pour lui-même, et accède au même espace cours/devoirs/progression que l'élève. Zéro duplication de code.
**Dépend de :** step_8.
**Branche :** `feat/etudiant-dashboard`

---

## Ce que tu livres à la fin de cette étape

- Dashboard `/dashboard/etudiant` avec ses pages propres (profs liés, paiements)
- Réutilisation complète de `/dashboard/eleve/*` pour cours, devoirs, ressources, progression
- `PaymentsTable` avec historique et bouton remboursement conditionnel

---

## Ordre d'exécution

### 1. Vérifier les routes existantes

Avant de coder quoi que ce soit, vérifier que les routes suivantes fonctionnent déjà pour le rôle ETUDIANT :
- `/dashboard/eleve/*` : le layout et chaque page vérifient `role === 'ELEVE' || role === 'ETUDIANT'`
- `/booking/[profId]` : la route de réservation accepte ETUDIANT
- `/api/bookings/create` : vérifie PARENT ou ETUDIANT, et dans ce cas `eleveId === session.user.id`
- `/api/join/[code]` : accepte ETUDIANT (dans ProfParent, `parentId = etudiantId`)

Corriger ces vérifications dans les fichiers existants si nécessaire. Ne pas dupliquer les pages.

### 2. Layout étudiant

Crée `/app/(auth)/dashboard/etudiant/layout.tsx`.

Vérifie `role === 'ETUDIANT'` côté serveur.
Affiche `EtudiantSidebar` avec deux sections :
- "Mon espace" : liens vers `/dashboard/eleve` (cours), `/dashboard/eleve/history`, `/dashboard/eleve/homework`, `/dashboard/eleve/resources`, `/dashboard/eleve/progress`
- "Réservations" : Mes professeurs (`/dashboard/etudiant/profs`), Mes paiements (`/dashboard/etudiant/payments`)

### 3. Page d'accueil étudiant

`/dashboard/etudiant/page.tsx` → redirect vers `/dashboard/eleve`.
Pas de contenu propre — la page de cours est la même.

### 4. Mes professeurs

Crée `GET /api/etudiant/profs` :
- Récupère les `ProfParent` où `parentId = session.user.id` (ETUDIANT)
- Pour chaque prof : `id`, `username`, `image`, `hourlyRate`, `subjects`, `bio`
- Compter le nombre de bookings et récupérer le prochain cours

Page `/dashboard/etudiant/profs` :
- Grille de cartes profs avec bouton "Réserver" → `/booking/[profId]`
- Si aucun prof lié : message avec instructions pour rejoindre via un code d'invitation

### 5. Mes paiements

Crée `GET /api/etudiant/payments` :
- Payments où `booking.payerId = session.user.id`
- Paginé (20 par page)
- Retourner : `id`, `amount`, `status`, `createdAt`, `refundedAt`, `booking.subject`, `booking.startTime`, `booking.status`, `booking.prof.username`

Page `/dashboard/etudiant/payments` :
- Tableau paginé des paiements
- Montants en euros (conversion depuis centimes dans le composant)
- Statut avec badge coloré : `EN_ATTENTE` (jaune), `REVERSE` (vert), `REMBOURSE` (rouge)
- Bouton "Annuler" visible uniquement si `payment.status === 'EN_ATTENTE'` et `booking.status === 'CONFIRME'` et cours dans le futur — appelle `POST /api/bookings/[bookingId]/refund`

### 6. Middleware

Vérifier que le middleware laisse passer ETUDIANT sur `/dashboard/eleve/*` et `/booking/*`.
Si ce n'est pas le cas → corriger les conditions du middleware (pas de nouveau fichier — modifier `/middleware.ts`).

---

## KPI de validation

- ETUDIANT → `/dashboard/etudiant` → redirect vers `/dashboard/eleve`
- ETUDIANT → `/dashboard/eleve/homework` → devoirs affichés (même page que ELEVE)
- ETUDIANT → `/booking/[profId]` → calendrier de réservation, `eleveId` auto-rempli avec son propre ID
- Paiement complété → visible dans `/dashboard/etudiant/payments`
- Bouton "Annuler" sur un paiement `REVERSE` → absent (pas affiché)
- `grep -rn "role === 'ELEVE'" ./app ./lib` → toutes les occurrences incluent `|| role === 'ETUDIANT'` ou utilisent un helper commun
- Zéro page dupliquée entre `/dashboard/eleve/*` et `/dashboard/etudiant/*`
- `tsc --noEmit` et `eslint` → 0 erreur

---

## Rappels architecturaux

- Open/Closed Principle : étendre le comportement des pages existantes par des guards de rôle, pas en copiant les fichiers
- L'étudiant est son propre "parent" dans la relation `ProfParent` — c'est un choix d'architecture intentionnel, pas un bug
- Toutes les pages `/dashboard/eleve/*` sont partagées avec ETUDIANT sans aucune modification de leur contenu
