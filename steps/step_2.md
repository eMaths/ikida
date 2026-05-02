# STEP 2 — Onboarding professeur

**Objectif :** un PROF nouvellement inscrit complète 4 étapes. Tant que c'est incomplet, toutes les routes `/dashboard/prof/*` redirigent vers l'onboarding. Quand c'est terminé, `onboardingCompleted = true`.
**Dépend de :** step_1.
**Branche :** `feat/prof-onboarding`

---

## Ce que tu livres à la fin de cette étape

- Tunnel onboarding 4 étapes avec progression visuelle
- `ProfProfile` créé en base avec toutes les infos
- Compte Stripe Connect Express lié
- Google Calendar OAuth avec scope calendar accordé
- `onboardingCompleted = true` après l'étape 4

---

## Ordre d'exécution

### 1. Clients externes

Crée `/lib/stripe.ts` : instance Stripe singleton avec la clé secrète depuis `process.env`.
Crée `/lib/r2.ts` : client S3 compatible Cloudflare R2 (endpoint R2, credentials depuis `process.env`).
Ces deux fichiers n'exposent aucun secret — uniquement des instances.

### 2. API Route upload fichiers

Crée `POST /api/upload`.

Ordre des opérations :
1. Vérifier la session (authentifié)
2. Lire le fichier depuis `FormData`
3. Vérifier le type MIME **côté serveur** (pas l'extension) — accepter seulement `image/jpeg`, `image/png`, `image/webp`
4. Vérifier la taille : max 5 Mo pour les avatars
5. Générer une clé unique `{type}/{userId}/{uuid}.{ext}`
6. Uploader vers R2 avec `PutObjectCommand`
7. Retourner l'URL publique

Cette route sera réutilisée pour les devoirs et ressources (les types MIME et tailles max seront différents).

### 3. Étape 1 — Informations personnelles

Crée `POST /api/prof/onboarding/profile`.

Reçoit : `username`, `bio`, `hourlyRate` (en centimes — la conversion euros→centimes se fait côté client avant l'envoi), `subjects[]`, `levels[]`, `imageUrl`.

Ordre :
1. Session + rôle PROF
2. Validation Zod (hourlyRate : entier, min 100 centimes, max 50000 centimes)
3. `User.update` pour username et image
4. `ProfProfile.upsert` pour les autres champs

### 4. Étape 2 — Google Calendar

L'OAuth Google est déjà configuré dans Auth.js avec le scope `calendar`.
Cette étape vérifie que le `refresh_token` est présent dans la table `Account`.

Crée `GET /api/prof/onboarding/check-calendar` :
- Vérifie que `Account` du prof contient un `refresh_token` Google et le scope `calendar`
- Retourne `{ connected: boolean, email: string }`

Dans le callback `jwt` d'Auth.js : après un OAuth Google d'un PROF, mettre à jour `ProfProfile.googleCalendarEmail`.

Côté UI : bouton "Connecter Google Calendar" → `signIn('google', { callbackUrl: '/dashboard/prof/onboarding' })`.
Le bouton "Suivant" est désactivé tant que `connected = false`.

### 5. Étape 3 — Stripe Connect

Crée `POST /api/stripe/connect/onboard` :
1. Session + rôle PROF
2. Si pas de `stripeAccountId` sur `ProfProfile` → créer un compte Stripe Connect Express via l'API Stripe, stocker l'ID
3. Générer un lien `accountLinks.create` avec `return_url` et `refresh_url` pointant vers l'onboarding
4. Retourner l'URL

Crée `GET /api/stripe/connect/status` :
- Récupère le compte Stripe via `stripe.accounts.retrieve`
- Retourne `{ connected: boolean }` selon `details_submitted && charges_enabled`

Le bouton "Suivant" est désactivé tant que `connected = false`.

### 6. Étape 4 — Politique de remboursement

Crée `POST /api/prof/onboarding/complete`.

Reçoit : `refundPolicy` (enum : `REFUND_2H`, `REFUND_24H`, `REFUND_48H`, `NO_REFUND`), `refundPolicyNote` (optionnel, max 300 chars).

Ordre :
1. Session + rôle PROF
2. Validation Zod
3. `ProfProfile.update` avec la politique
4. `User.update` : `onboardingCompleted = true`
5. Retourner 200

Après succès → redirection vers `/dashboard/prof`.

### 7. Page onboarding

Page unique `/dashboard/prof/onboarding` avec une barre de progression 4 étapes.
L'état de l'étape courante est géré côté client.
Chaque étape est un composant séparé, responsabilité unique.

---

## KPI de validation

- PROF qui se connecte avec `onboardingCompleted = false` → redirigé automatiquement vers l'onboarding
- Étape 1 : `SELECT * FROM "ProfProfile"` → 1 ligne avec `hourlyRate` en centimes (ex: 3000 pour 30€, pas 30.0)
- Étape 2 : `SELECT refresh_token FROM "Account" WHERE provider = 'google'` → non null
- Étape 3 : `SELECT "stripeAccountId" FROM "ProfProfile"` → commence par `acct_`
- Étape 4 : `SELECT "refundPolicy", "onboardingCompleted" FROM "ProfProfile" JOIN "User"` → valeurs correctes
- Après étape 4 : accéder à `/dashboard/prof` → dashboard affiché (plus de redirect onboarding)
- Upload photo : fichier > 5 Mo → rejeté côté serveur avec 400
- `tsc --noEmit` et `eslint` → 0 erreur

---

## Rappels architecturaux

- `stripeAccountId` et tokens OAuth ne sont jamais loggués
- Le `hourlyRate` est **toujours** un entier en centimes — la conversion en euros n'existe que dans l'affichage
- Les étapes 2 et 3 dépendent de services externes — prévoir un message clair si le service est indisponible
- Zéro logique Stripe ou Google Calendar dans les composants React
