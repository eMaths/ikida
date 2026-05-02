# STEP 0 — Socle technique

**Rôle de cette étape :** poser les fondations. Rien d'autre ne peut démarrer sans ça.
**Branche :** `chore/setup`
**Merge dans :** `main` une fois tous les KPI verts.

---

## Ce que tu livres à la fin de cette étape

- Projet Next.js 15 App Router qui compile sans erreur
- Base PostgreSQL Neon connectée, schéma Prisma migré intégralement
- Auth.js v5 opérationnel (Google OAuth + Credentials)
- 7 badges seedés en base
- Variables d'environnement documentées

---

## Ordre d'exécution

### 1. Initialisation du projet

Crée le projet Next.js 15 avec TypeScript strict, ESLint, Tailwind, App Router.
Installe les dépendances du projet dès maintenant, pas au fil de l'eau :
`prisma`, `@prisma/client`, `next-auth@beta`, `bcryptjs`, `zod`, `lucide-react`,
`@aws-sdk/client-s3`, `stripe`, `resend`, `shadcn/ui`.

Configure `tsconfig.json` avec `strict: true` et `noUncheckedIndexedAccess: true`.
Configure ESLint pour interdire `any`, `console.log` et les imports non utilisés.

**Nom de l'app :** Ikida — "Apprenez mieux, gérez moins"

### 2. Schéma Prisma

Source de vérité : `SPEC BDD.md`. Retranscris **intégralement** le schéma dans `schema.prisma`.
Aucune liberté sur les noms de champs, les types, les relations.
Points d'attention :
- Tous les IDs sont des UUID (`@default(uuid())`)
- `hourlyRate`, `amount`, `platformFee`, `profAmount` : toujours `Int` (centimes)
- `RefundPolicy` : 4 valeurs — `REFUND_2H`, `REFUND_24H`, `REFUND_48H`, `NO_REFUND`
- `SessionLog` : pas de champ `ipAddress` — RGPD
- `EleveProgress` : contient `pendingBadgeSlug` pour la modale gamification

Lance `prisma migrate dev --name init` sur une base vide.
Vérifie que `prisma migrate status` indique tout appliqué.

### 3. Client Prisma singleton

Crée `/lib/prisma.ts` : instance Prisma singleton (pattern standard Next.js dev/prod).
Ce fichier est le seul endroit où Prisma est instancié dans tout le projet.

### 4. Auth.js v5

Crée `/lib/auth.ts` avec :
- Provider `Credentials` : récupère l'utilisateur par email, vérifie bcrypt, vérifie `bannedAt`
- Provider `Google` avec scope `calendar` (nécessaire pour l'onboarding prof)
- Callback `jwt` : ajoute `id`, `role`, `username`, `onboardingCompleted` dans le token
- Callback `session` : expose ces champs sur `session.user`
- Callback `signIn` : bloque si `bannedAt` non null, insère un `SessionLog` (sans IP, avec `userAgent`)

Crée `/app/api/auth/[...nextauth]/route.ts` qui exporte les handlers Auth.js.

### 5. Middleware

Crée `/middleware.ts`. Logique :
- Route publique (`/`, `/login`, `/register`, `/join/*`) → laisser passer
- Route `/dashboard/prof/*` sans session → redirect `/login`
- Route `/dashboard/prof/*` avec `onboardingCompleted = false` → redirect `/dashboard/prof/onboarding`
- Route `/admin/*` sans rôle ADMIN → redirect `/login`
- Chaque rôle ne peut accéder qu'à son propre dashboard

### 6. Variables d'environnement

Crée `.env.example` avec toutes les clés nécessaires (sans valeurs).
Crée `.env.local` (gitignored) avec tes vraies valeurs de dev.
Clés requises : `DATABASE_URL`, `NEXTAUTH_SECRET`, `NEXTAUTH_URL`,
`GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `STRIPE_SECRET_KEY`,
`STRIPE_WEBHOOK_SECRET`, `STRIPE_PUBLISHABLE_KEY`, `PLATFORM_FEE_PERCENT`,
`RESEND_API_KEY`, `RESEND_FROM_EMAIL`, `R2_ACCOUNT_ID`, `R2_ACCESS_KEY_ID`,
`R2_SECRET_ACCESS_KEY`, `R2_BUCKET_NAME`, `R2_PUBLIC_URL`, `CRON_SECRET`.

### 7. Design system

Configure Tailwind avec la palette Ikida dans `tailwind.config.ts` :
- Primaire : `#6C63FF`
- Secondaire : `#48CAE4`
- Succès : `#6BCB77`
- Erreur : `#EF5350`
- Accent : `#FFD166`
- Fond : `#F7F8FC`
- Texte principal : `#1A1A2E`

Configure shadcn/ui. Police : Nunito (Google Fonts).
Règle absolue UI : uniquement des icônes `lucide-react`, zéro emoji.

### 8. Seed badges

Crée `/prisma/seed.ts`. Insère les 7 badges définis dans le tableau de `step_8.md §2` (slugs, labels, icônes Lucide) avec `upsert` (idempotent).
C'est le seul seed de toute la plateforme — les badges sont des données statiques référencées par slug.
Configure `prisma.seed` dans `package.json`.

### 9. Layout racine et page placeholder

Crée `/app/layout.tsx` avec : police Nunito, métadonnées Ikida, fond `#F7F8FC`.
Crée `/app/(public)/page.tsx` : page de garde minimaliste avec le nom "Ikida",
le slogan "Apprenez mieux, gérez moins", et deux boutons Se connecter / Créer un compte.
Rien d'autre — cette page sera enrichie après l'auth.

---

## KPI de validation — tous doivent être verts avant de passer à step_1

- `npx tsc --noEmit` → 0 erreur
- `npx eslint .` → 0 erreur, 0 warning
- `npx prisma migrate status` → toutes migrations appliquées
- `npx prisma db seed` → "7 badges insérés" sans erreur
- `SELECT slug FROM "Badge"` → 7 lignes
- `SELECT * FROM "SessionLog"` → table vide mais existante
- `GET http://localhost:3000/api/auth/session` → réponse JSON (vide ou session)
- Connexion Google OAuth sur `/login` → `User` et `Account` créés en base
- Page `/` accessible sans session → affiche "Ikida"
- `SELECT "passwordHash" FROM "User" LIMIT 1` → commence par `$2b$12$`

---

## Rappels architecturaux

- Zéro logique métier dans les composants — uniquement dans les API Routes et `/lib`
- Zéro `any` TypeScript
- Zéro `console.log` — uniquement `console.error` dans les blocs `catch`
- Tous les montants financiers sont des entiers (centimes), jamais des floats
- `SessionLog` ne contient jamais d'IP
