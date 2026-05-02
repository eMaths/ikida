# ROADMAP DE DÉVELOPPEMENT

> Ordre strict. On ne commence pas une étape sans avoir validé tous les KPI de l'étape précédente.
> Chaque jalon donne lieu à un rapport agent (format défini dans AGENT_RULES.md section 13) et à un commit sur une branche dédiée.

---

## ÉTAPE 0 — Socle technique
> **Objectif :** le projet tourne localement, la base de données est accessible, Auth.js répond.
> **Durée estimée :** 1 session.

### Sous-jalons
- [ ] 0.1 — Init Next.js 15 + TypeScript strict + ESLint + Prettier
- [ ] 0.2 — Prisma 5 configuré, `DATABASE_URL` Neon connecté
- [ ] 0.3 — Schéma Prisma complet généré depuis SPEC BDD.md (toutes les tables)
- [ ] 0.4 — `prisma migrate dev` passe sans erreur sur base vide
- [ ] 0.5 — `prisma db seed` peuple les badges (catalogue initial — **seul seed de toute la plateforme**, justifié : les badges sont des données statiques référencées par slug dans le code, sans elles `checkAndAwardBadges()` échoue au runtime)
- [ ] 0.6 — Auth.js v5 installé, provider Google + Credentials configurés
- [ ] 0.7 — Variables d'environnement : `.env.local` + `.env.example` créés

### KPI de validation
- `tsc --noEmit` : 0 erreur
- `eslint` : 0 warning, 0 erreur
- `prisma migrate status` : toutes les migrations appliquées
- `prisma db seed` : 7 badges insérés, 0 erreur
- `GET /api/auth/session` depuis un navigateur : réponse JSON (même vide)
- Connexion Google OAuth complète sur `/login` : session créée en base (table `Account` et `User`)

---

## ÉTAPE 1 — Authentification & Inscription
> **Objectif :** n'importe quel rôle peut créer un compte et se connecter. Les CGU sont acceptées. Le `SessionLog` est alimenté sans IP.
> **Dépend de :** Étape 0 complète.

### Sous-jalons
- [ ] 1.1 — Page `/register` : formulaire username + email + mdp + rôle + CGU
- [ ] 1.2 — API Route `POST /api/auth/register` avec validation Zod
- [ ] 1.3 — Page `/login` : email/mdp + OAuth Google
- [ ] 1.4 — Middleware Auth.js : redirection par rôle après connexion
- [ ] 1.5 — `SessionLog` inséré à chaque connexion (sans IP, avec userAgent)
- [ ] 1.6 — Page de garde `/` : redirection si connecté, bouton sinon
- [ ] 1.7 — Bannissement : `bannedAt` vérifié au login, accès bloqué

### KPI de validation
- Créer un compte PROF → redirigé vers `/dashboard/prof/onboarding`
- Créer un compte PARENT → redirigé vers `/dashboard/parent`
- Email déjà existant → message d'erreur, pas de doublon en base
- OAuth Google avec email inconnu → message d'erreur, pas de création automatique
- Mauvais mot de passe → message générique (pas de distinction email/mdp)
- Utilisateur banni → connexion refusée
- Vérifier en base : `SessionLog` créé, champ `userAgent` présent, **pas de champ IP**
- `User` en base : `username` renseigné, `role` correct, `passwordHash` hashé (bcrypt)

---

## ÉTAPE 2 — Onboarding professeur
> **Objectif :** un PROF peut compléter les 4 étapes. Tant que non terminé, toutes les routes `/dashboard/prof/*` redirigent vers l'onboarding.
> **Dépend de :** Étape 1 complète.

### Sous-jalons
- [ ] 2.1 — Tunnel 4 étapes avec barre de progression
- [ ] 2.2 — Étape 1 : formulaire profil → `ProfProfile` créé en base
- [ ] 2.3 — Étape 2 : OAuth Google Calendar → tokens stockés chiffrés dans `Account`
- [ ] 2.4 — Étape 3 : Stripe Connect Express → `stripeAccountId` stocké dans `ProfProfile`
- [ ] 2.5 — Étape 4 : politique de remboursement → `refundPolicy` + `refundPolicyNote` en base
- [ ] 2.6 — `onboardingCompleted = true` après étape 4, débloque le dashboard
- [ ] 2.7 — Middleware : toute route `/dashboard/prof/*` sans `onboardingCompleted` → redirect

### KPI de validation
- Accéder à `/dashboard/prof` sans onboarding terminé → redirect vers `/dashboard/prof/onboarding`
- Étape 2 : bouton Suivant désactivé si Google Calendar non connecté
- Étape 3 : bouton Suivant désactivé si `stripeAccountId` absent en base
- Étape 4 terminée → `onboardingCompleted = true` en base, accès dashboard débloqué
- Vérifier en base : `ProfProfile` complet, `Account` avec `access_token` et `refresh_token` non nuls
- Tarif à 0 ou > 500 → message d'erreur, pas de sauvegarde

---

## ÉTAPE 3 — Dashboard admin
> **Objectif :** l'admin peut lister les utilisateurs, consulter les logs de connexion, bannir un utilisateur.
> **Dépend de :** Étape 1 complète.

### Sous-jalons
- [ ] 3.1 — Page `/admin` : liste des utilisateurs paginée (25/page), filtres rôle/statut
- [ ] 3.2 — Modale détail utilisateur + bouton Bannir (avec confirmation)
- [ ] 3.3 — Page `/admin/logs` : tableau SessionLog paginé (50/page), sans colonne IP
- [ ] 3.4 — Page `/admin/transactions` : tableau Payment (lecture seule)

### KPI de validation
- Accéder à `/admin/*` sans rôle ADMIN → redirect `/login`
- Bannir un utilisateur → `bannedAt` rempli en base, l'utilisateur ne peut plus se connecter
- Colonne IP absente de la page logs (vérifier DOM et requête Prisma)
- Tableau transactions : statuts `EN_ATTENTE / REVERSE / REMBOURSE` affichés correctement

---

## ÉTAPE 4 — Lien prof ↔ parent + page réservation (sans paiement)
> **Objectif :** un parent peut rejoindre un prof via code d'invitation. Le calendrier des dispos s'affiche. Aucun paiement encore.
> **Dépend de :** Étapes 1 et 2 complètes.

### Sous-jalons
- [ ] 4.1 — PROF-08 : génération du code d'invitation (`inviteCode` + `inviteCodeExpiresAt` dans `ProfProfile`)
- [ ] 4.2 — Route `/join/[code]` : lie le parent au prof via `ProfParent`
- [ ] 4.3 — PROF-03 : page disponibilités → écriture en base dans `ProfAvailability` (pas dans Google Calendar)
- [ ] 4.4 — PAR-02 : section "Mes professeurs" dans le dashboard parent
- [ ] 4.5 — PAR-03 : page booking → calcul des créneaux libres depuis `ProfAvailability` croisiés avec les `Booking` existants
- [ ] 4.6 — Sélection de créneaux + choix de l'élève → récapitulatif (sans paiement encore)
- [ ] 4.7 — PAR-04 : "Ajouter un enfant" → création compte ELEVE + envoi EMAIL-07

### KPI de validation
- Code expiré (> 7 jours) → message d'erreur, pas de liaison
- Code déjà utilisé → "Vous êtes déjà lié à ce professeur"
- Sauvegarde dispos → lignes `ProfAvailability` créées (vérifier en base)
- `GET /api/prof/[profId]/slots` : aucun appel Google Calendar (purement BDD)
- Calendrier parent : créneaux libres affichés, créneaux déjà bookés exclus
- `ProfParent` créé en base avec les bons IDs

---

## ÉTAPE 5 — Flux de paiement Stripe
> **Objectif :** un parent peut payer un ou plusieurs créneaux. Le booking est créé. L'événement Google Calendar avec Meet est généré.
> **Dépend de :** Étape 4 complète.

### Sous-jalons
- [ ] 5.1 — API Route `POST /api/bookings/create` : vérification dispo + Stripe Checkout Session
- [ ] 5.2 — Webhook `POST /api/webhooks/stripe` : vérification signature + création `Booking` + `Payment`
- [ ] 5.3 — GCal-02 : création événement Google Calendar + lien Meet → `meetLink` en base
- [ ] 5.4 — Emails EMAIL-01 (parent) + EMAIL-02 (prof) envoyés après confirmation
- [ ] 5.5 — PAY-03 : remboursement selon politique prof
- [ ] 5.6 — Cron `GET /api/cron/payouts` (toutes les heures) : bascule applicative `EN_ATTENTE → REVERSE` 24h après la fin du cours + EMAIL-04 (PAS d'appel Stripe Transfer — destination charges)
- [ ] 5.7 — Cron quotidien `GET /api/cron/reminders` (08h00) : EMAIL-03 J-1

### KPI de validation
- Paiement Stripe test réussi → `Booking` en base avec statut `CONFIRME`, `Payment` avec montants en centimes
- Vérifier : `platformFee` = montant × `PLATFORM_FEE_PERCENT` / 100 (entier, pas de float)
- `meetLink` non nul en base après webhook
- Webhook avec signature invalide → `400` immédiat, aucun traitement
- Double-réservation (même créneau) → remboursement automatique + email parent
- Remboursement hors délai politique → `403` avec message
- Cron payout : `payoutSentAt` rempli une seule fois par booking (idempotence via condition `status = 'EN_ATTENTE'`). Aucun appel `stripe.transfers.create` dans le code.
- Cron protégé : appel sans header `CRON_SECRET` → `401`
- Tous les montants : 0 float dans la base (vérifier les champs `amount`, `platformFee`, `profAmount`)

---

## ÉTAPE 6 — Dashboard élève : cours, devoirs, ressources
> **Objectif :** l'élève accède à ses cours, rend ses devoirs, consulte ses ressources.
> **Dépend de :** Étape 5 complète.

### Sous-jalons
- [ ] 6.1 — Gate "aucun cours à venir" → page de blocage exclusive
- [ ] 6.2 — ELV-01 : cours à venir + logique bouton Meet (−15min / +1h)
- [ ] 6.3 — ELV-02 : cours passés + indicateur devoir
- [ ] 6.4 — PROF-06 : création devoirs par le prof + upload fichiers (HomeworkFile → R2)
- [ ] 6.5 — ELV-03 : rendu devoir (texte + fichiers) + EMAIL-09 au prof
- [ ] 6.6 — Correction devoir par le prof (CORRIGE) → visible élève
- [ ] 6.7 — PROF-06 : ajout ressource (Resource + ResourceFile) → ELV-04

### KPI de validation
- Élève sans cours à venir → aucune autre page accessible (tester toutes les routes directement)
- Bouton Meet : inactif à J-1, actif 15min avant, inactif 1h après la fin
- Rendu après date limite → zone grisée, API refuse (`403`)
- Devoir rendu → statut `RENDU` en base + EMAIL-09 envoyé
- Prof corrige → statut `CORRIGE` en base, correction visible par l'élève
- Upload fichier > 10 Mo → refusé côté serveur (pas seulement client)
- Upload > 3 fichiers → refusé côté serveur

---

## ÉTAPE 7 — Questionnaires post-séance + emails restants
> **Objectif :** prof et élève remplissent leurs questionnaires. `attendance` est enregistré. Les emails de réunion et de rappel fonctionnent.
> **Dépend de :** Étape 6 complète.

### Sous-jalons
- [ ] 7.1 — Cron : booking `endTime + 15min` passé → statut `TERMINE` + notification prof (PROF-07)
- [ ] 7.2 — PROF-07 : formulaire post-séance → `PostSeanceProf` en base (avec `attendance`)
- [ ] 7.3 — `needsMeeting = true` → EMAIL-08 envoyé au parent
- [ ] 7.4 — ELV-05 : bannière questionnaire élève (1 fois par cours) → `PostSeanceEleve` en base
- [ ] 7.5 — Dashboard parent PAR-04 : affichage scores prof (hors `notesPrivees`)

### KPI de validation
- Questionnaire prof : `attendance` obligatoire avant les scores
- `notesPrivees` : requête Prisma côté parent/élève n'inclut **jamais** ce champ (vérifier le `select`)
- `needsMeeting = true` → EMAIL-08 envoyé, badge rouge visible sur fiche enfant côté parent
- ELV-05 : questionnaire proposé exactement une fois par cours, "Passer" = aucune relance
- `PostSeanceProf` : 1 seule entrée par `bookingId` (contrainte `@unique` en base)

---

## ÉTAPE 8 — Gamification
> **Objectif :** XP, badges et streaks fonctionnent de bout en bout. La page suivi s'affiche avec les vrais graphiques.
> **Dépend de :** Étape 7 complète.

### Sous-jalons
- [ ] 8.1 — `lib/gamification.ts` : fonctions `addXpEvent(eleveId, type, referenceId)`, `checkAndAwardBadges(eleveId)`, `calculateWeeklyStreak(eleveId)`, `calculateLevel(totalXp)` (cf. step_8)
- [ ] 8.2 — Branchement XP sur les 5 déclencheurs (cron TERMINE, rendu devoir, questionnaire élève, éval prof, streak)
- [ ] 8.3 — `EleveProgress` créé automatiquement à la création d'un compte ELEVE/ETUDIANT
- [ ] 8.4 — `pendingBadgeSlug` → modale de félicitations à la prochaine page vue, puis remis à `null`
- [ ] 8.5 — ELV-06 / GAM-01 : page suivi — barre XP, niveaux, graphique XP/temps, graphique scores, grille badges, streak
- [ ] 8.6 — PAR-04 : lecture seule du suivi gamifié de l'enfant

### KPI de validation
- `EleveProgress.totalXp` == somme de tous les `XpEvent.xpGained` de l'élève (vérifier en base)
- Un même booking ne déclenche `COURS_EFFECTUE` qu'une seule fois (tester appel double)
- Un même devoir ne déclenche `DEVOIR_RENDU` qu'une seule fois
- Badge "Premier pas" attribué après le 1er cours → `EleveBadge` en base + modale affichée
- Graphiques s'affichent avec 1 seul point de données (pas de crash)
- `notesPrivees` absent des données retournées par l'API pour le graphique scores (vérifier la réponse JSON)
- Icônes badges : noms Lucide uniquement, aucun emoji dans le DOM

---

## ÉTAPE 9 — Dashboard étudiant autonome
> **Objectif :** l'étudiant autonome a un dashboard fonctionnel couvrant réservation + suivi.
> **Dépend de :** Étapes 5, 6, 7, 8 complètes.

### Sous-jalons
- [ ] 9.1 — Routes `/dashboard/etudiant/*` : reprendre les composants existants (pas de duplication)
- [ ] 9.2 — Réservation sans sélecteur d'élève (il réserve pour lui-même)
- [ ] 9.3 — Suivi gamifié identique à ELV-06

### KPI de validation
- `Booking.eleveId == Booking.payerId` pour un étudiant (vérifier en base)
- Toutes les features ELEVE + PARENT fonctionnent pour le rôle ETUDIANT
- Aucun composant dupliqué (réutilisation des composants existants)

---

## ÉTAPE 10 — Recette finale & sécurité
> **Objectif :** tous les critères de la checklist du CDCF (section 14) sont cochés. Aucune faille de sécurité.
> **Dépend de :** Étapes 0 à 9 complètes.

### Sous-jalons
- [ ] 10.1 — Parcourir toute la checklist CDCF section 14, cocher chaque item
- [ ] 10.2 — Vérifier chaque API Route : session → rôle → Zod → logique (ordre AGENT_RULES section 6)
- [ ] 10.3 — Vérifier toutes les requêtes Prisma : `notesPrivees` jamais exposé, pas de `findMany` sans `select`
- [ ] 10.4 — Vérifier en-têtes HTTP (`X-Frame-Options`, `X-Content-Type-Options`, etc.) dans `next.config.ts`
- [ ] 10.5 — `tsc --noEmit` + `eslint` : 0 erreur, 0 warning
- [ ] 10.6 — Aucun `console.log` de debug restant (grep `console.log` dans `/app` et `/lib`)
- [ ] 10.7 — `.env.example` à jour avec toutes les variables
- [ ] 10.8 — `README.md` : prérequis, variables, commandes de lancement, migration, seed

### KPI de validation (recette finale)
- Tous les items de la checklist CDCF section 14 : cochés
- `grep -r "console.log" ./app ./lib` : 0 résultat
- `grep -r "any" ./app ./lib --include="*.ts" --include="*.tsx"` : 0 résultat
- `grep -r "TODO\|FIXME" ./app ./lib` : 0 résultat
- Déploiement Vercel preview : build passe sans erreur
- Test de bout en bout manuel du flux critique : inscription prof → onboarding → code invitation → réservation parent → paiement → cours → questionnaires → XP + badge

---

## Récapitulatif des jalons

| Étape | Livrable principal | Dépend de |
| :---- | :---- | :---- |
| 0 | Socle technique + BDD + Auth.js | — |
| 1 | Inscription + connexion tous rôles | 0 |
| 2 | Onboarding prof complet | 1 |
| 3 | Dashboard admin | 1 |
| 4 | Liaison prof-parent + calendrier | 1, 2 |
| 5 | Paiement Stripe + webhooks + Meet | 4 |
| 6 | Dashboard élève : cours, devoirs, ressources | 5 |
| 7 | Questionnaires post-séance | 6 |
| 8 | Gamification XP + badges | 7 |
| 9 | Dashboard étudiant autonome | 5, 6, 7, 8 |
| 10 | Recette finale + sécurité | 0–9 |

> **Règle absolue :** ne jamais démarrer une étape dont une dépendance n'a pas tous ses KPI validés. En cas de doute sur un KPI, demander à l'humain avant de continuer.
