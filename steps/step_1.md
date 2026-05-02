# STEP 1 — Authentification & inscription

**Objectif :** n'importe quel rôle peut créer un compte, se connecter, et être redirigé vers son dashboard. Les utilisateurs bannis sont bloqués.
**Dépend de :** step_0 — tous les KPI verts.
**Branche :** `feat/auth`

---

## Ce que tu livres à la fin de cette étape

- Pages `/login` et `/register` fonctionnelles
- API Route `POST /api/auth/register` sécurisée
- SessionLog alimenté à chaque connexion (sans IP)
- Redirection par rôle après connexion
- Bannissement bloquant

---

## Ordre d'exécution

### 1. API Route d'inscription

Crée `POST /api/auth/register`.

Ordre des opérations dans la route (respecter cet ordre) :
1. Parser et valider le body avec Zod : `username`, `email`, `password`, `role`, `acceptedCgu`, `inviteCode` (optionnel)
2. Rate limiting : 5 tentatives / 10 min par IP (utilise le header `x-forwarded-for`)
3. Vérifier unicité de l'email
4. Hasher le mot de passe avec bcrypt, cost factor 12
5. Créer le `User` avec `cguAcceptedAt = now()` (preuve de consentement RGPD)
6. Si rôle `ELEVE` ou `ETUDIANT` → créer `EleveProgress` lié
7. Si rôle `PROF` → `onboardingCompleted = false`
8. Retourner 201 avec `{ userId, role }`

Règles :
- Validation Zod : username 3-30 chars alphanumérique, mot de passe min 8 chars + 1 majuscule + 1 chiffre
- En cas d'email existant → 409, message générique (pas de confirmation d'existence)
- `acceptedCgu` doit être `true` — sinon 400

### 2. Page d'inscription `/register`

Formulaire : username, email, mot de passe, confirmation, sélecteur de rôle (4 options visuelles), case CGU.
Si rôle `ELEVE` → afficher un champ optionnel "code d'invitation".

Comportement :
- Validation légère côté client pour l'UX (erreurs immédiates)
- Validation définitive côté serveur — ne jamais faire confiance au client
- Après inscription réussie → `signIn('credentials')` automatique → redirection selon rôle :
  - PROF → `/dashboard/prof/onboarding`
  - PARENT → `/dashboard/parent`
  - ELEVE → `/dashboard/eleve`
  - ETUDIANT → `/dashboard/etudiant`

### 3. Page de connexion `/login`

Formulaire : email, mot de passe.
Bouton Google OAuth séparé.

Comportements :
- Mauvais identifiants → message générique "Email ou mot de passe incorrect" (jamais distinguer lequel est faux)
- Compte banni → message "Votre compte a été suspendu"
- Google OAuth → si email inconnu → créer le compte avec rôle à choisir (ou bloquer ? → demander à l'humain)
- `callbackUrl` en query param → rediriger vers cette URL après connexion

### 4. Dashboards placeholder

Crée une page placeholder pour chaque dashboard :
`/dashboard/prof`, `/dashboard/parent`, `/dashboard/eleve`, `/dashboard/etudiant`, `/admin`.

Chaque placeholder affiche juste le username et le rôle de la session.
Ces pages seront remplacées dans les étapes suivantes.
Le layout de chaque dashboard vérifie la session côté serveur (doublon de sécurité avec le middleware).

---

## KPI de validation

- Inscription PROF → redirigé `/dashboard/prof/onboarding` (placeholder)
- Inscription PARENT → redirigé `/dashboard/parent`
- Email déjà pris → 409, pas de doublon en base
- Mauvais mot de passe → message générique, pas de précision
- Accéder à `/dashboard/prof` sans session → redirect `/login`
- Accéder à `/admin` avec un compte PARENT → redirect `/login`
- Après connexion → `SELECT * FROM "SessionLog"` → 1 ligne créée, colonne `ipAddress` inexistante
- Utilisateur avec `bannedAt` non null → connexion refusée
- `tsc --noEmit` et `eslint` → 0 erreur

---

## Rappels architecturaux

- Le `SessionLog` est créé dans le callback `signIn` d'Auth.js, pas dans la page
- Jamais de logique de redirection dans les composants — utiliser le middleware et les Server Components
- Les layouts de dashboard vérifient la session côté serveur en plus du middleware (défense en profondeur)
- Le rate limiting est côté serveur uniquement
