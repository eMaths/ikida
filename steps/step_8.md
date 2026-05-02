# STEP 8 — Gamification : XP, badges, streaks

**Objectif :** chaque action élève génère de l'XP. Les badges sont attribués automatiquement. Une modale s'affiche au déblocage. La page "Mon suivi" centralise tout.
**Dépend de :** step_7. Les TODO laissés dans step_5, step_6 et step_7 sont branchés ici.
**Branche :** `feat/gamification`

---

## Ce que tu livres à la fin de cette étape

- Moteur XP + badges dans `/lib/gamification.ts` (server-side only)
- `EleveProgress` mis à jour à chaque action
- Modale de déblocage de badge (icônes Lucide, zéro emoji)
- Page "Mon suivi" avec barre XP, streak, grille badges
- Tous les TODO de step_5, step_6, step_7 branchés

---

## Ordre d'exécution

### 1. Moteur de gamification

Crée `/lib/gamification.ts`.

Ce module est **importé uniquement dans des API Routes et des fonctions serveur** — jamais dans un composant client.

Fonctions à exposer :

**`addXpEvent(eleveId, type, referenceId)`**
- `referenceId` est **obligatoire** (clé d'idempotence). Exemples : `bookingId` pour `COURS_EFFECTUE`, `homeworkId` pour `DEVOIR_RENDU`, `"streak-2026-W18"` pour `STREAK_SEMAINE`.
- L'idempotence est garantie par la contrainte `@@unique([eleveId, type, referenceId])` en base : on tente l'insert, si erreur P2002 → retour immédiat (no-op).
- Crée le `XpEvent` et incrémente `EleveProgress.totalXp` en transaction atomique.
- Mappage type → XP (source de vérité : CDCF GAM-02) :
  - `COURS_EFFECTUE` → +10 XP
  - `DEVOIR_RENDU` → +15 XP
  - `DEVOIR_RENDU_EN_AVANCE` → +5 XP (bonus en plus de DEVOIR_RENDU)
  - `QUESTIONNAIRE_REMPLI` → +5 XP
  - `EVALUATION_PROF_POSITIVE` → +10 XP (une seule fois par cours)
  - `STREAK_SEMAINE` → +20 XP par pallier

**`checkAndAwardBadges(eleveId)`**
- Charge les badges déjà obtenus par l'élève
- Vérifie chaque condition de badge (voir tableau ci-dessous)
- Pour chaque badge débloqué pour la première fois : créer `EleveBadge`, et mettre `EleveProgress.pendingBadgeSlug` si null (ne pas écraser un badge déjà en attente d'affichage). **Pas de bonus XP pour le déblocage d'un badge** — le badge est sa propre récompense (cf. CDCF, pas de type `BADGE_DEBLOQUE`).

**`calculateWeeklyStreak(eleveId)`**
- Streak **hebdomadaire** (cf. CDCF GAM-01) : nombre de semaines ISO consécutives pendant lesquelles l'élève a eu au moins 1 booking `TERMINE` avec présence (`PostSeanceProf.attendance = true` si dispo, sinon TERMINE suffit).
- Algorithme : grouper les bookings TERMINE de l'élève par semaine ISO (`year-W##`), partir de la semaine en cours et reculer tant qu'il y a au moins 1 cours par semaine.
- Met à jour `EleveProgress.currentStreak` et `longestStreak` si record battu.
- Déclenche `addXpEvent(..., 'STREAK_SEMAINE', "streak-{year}-W{week}")` lorsqu'un nouveau pallier hebdo est franchi (referenceId garantit que chaque semaine n'attribue l'XP qu'une seule fois).

**`calculateLevel(totalXp)`**
- Paliers fixes (source : CDCF GAM-01) :
  - Débutant : 0–99
  - Curieux : 100–249
  - Explorateur : 250–499
  - Aventurier : 500–899
  - Expert : 900–1499
  - Maître : 1500+
- Retourne `{ name, current, nextThreshold }`.

### 2. Conditions de déblocage des 7 badges

Source de vérité : CDCF GAM-01. Slugs / labels / icônes Lucide :

| Slug | Label | Icône Lucide | Condition |
|---|---|---|---|
| `premier-pas` | Premier pas | `Sparkles` | 1 booking TERMINE avec présence |
| `assidu` | Assidu | `Calendar` | 5 bookings TERMINE avec présence |
| `regulier` | Régulier | `Repeat` | streak hebdomadaire ≥ 3 |
| `devoirs-faits` | Devoirs faits | `BookOpenCheck` | 5 devoirs en statut RENDU ou CORRIGE |
| `super-eleve` | Super élève | `Star` | sur 3 derniers `PostSeanceProf` consécutifs (présence true) : moyenne(`attitude`, `comprehension`, `confiance`) ≥ 4 |
| `streak-x5` | Streak ×5 | `Flame` | streak hebdomadaire ≥ 5 |
| `maitre` | Maître | `Trophy` | `EleveProgress.totalXp` ≥ 1500 |

Ces 7 badges sont la liste **exacte** seedée dans `prisma/seed.ts` (cf. step_0).

### 3. Brancher les TODO

Dans `/api/cron/complete-bookings` : après le `updateMany`, pour chaque booking passé en TERMINE :
- Si présence (PostSeanceProf.attendance true OU pas encore de questionnaire → on attribue par défaut, le prof régit plus tard) : `addXpEvent(eleveId, 'COURS_EFFECTUE', bookingId)`.
- Appeler `calculateWeeklyStreak(eleveId)` qui déclenche `STREAK_SEMAINE` si pallier franchi.
- Appeler `checkAndAwardBadges(eleveId)`.

Dans `/api/eleve/homework/[homeworkId]/submit` : après la mise à jour, appeler `addXpEvent(eleveId, 'DEVOIR_RENDU', homeworkId)` ; si `submittedAt < dueDate - 1h`, appeler aussi `addXpEvent(eleveId, 'DEVOIR_RENDU_EN_AVANCE', homeworkId)`. Puis `checkAndAwardBadges`.

Dans `/api/eleve/post-seance` : `addXpEvent(eleveId, 'QUESTIONNAIRE_REMPLI', bookingId)` puis `checkAndAwardBadges`.

Dans `/api/prof/post-seance` : si `attitude >= 4 OU comprehension >= 4`, appeler `addXpEvent(eleveId, 'EVALUATION_PROF_POSITIVE', bookingId)` puis `checkAndAwardBadges`.

### 4. API Routes gamification

Crée `GET /api/gamification/progress` : retourne pour l'élève connecté `{ totalXp, level: { name, current, nextThreshold }, streak, pendingBadgeSlug }`.

Crée `GET /api/gamification/badges` : retourne tous les badges avec `{ earned: boolean, unlockedAt }` pour l'élève connecté.

Crée `POST /api/gamification/pending-badge` : remet `pendingBadgeSlug = null` dans `EleveProgress`. Appelé par la modale après affichage.

### 5. Modale de déblocage

Dans le layout `/dashboard/eleve/layout.tsx` : charger `EleveProgress.pendingBadgeSlug`.
Si non null → passer à un composant client wrapper qui affiche la modale.

La modale :
- S'affiche une fois par badge, après fermeture elle ne réapparaît pas
- Icône du badge via `LucideIcons[badge.icon]` — résolution dynamique du nom stocké en base
- Animation subtile (pulse ou scale) — CSS uniquement, pas de lib d'animation externe
- Bouton fermeture → appelle `POST /api/gamification/pending-badge` → met l'état local à null

### 6. Page "Mon suivi"

Page `/dashboard/eleve/progress` (remplace le placeholder) :

- Barre de progression XP : niveau actuel, XP dans le niveau, XP requis pour le suivant
- Compteur streak avec icône `Flame` de Lucide
- Grille des 7 badges : badges obtenus en couleur, badges verrouillés en grisé avec icône `Lock`
- Résolution des icônes : `LucideIcons[badge.icon]` — si le nom n'est pas un composant Lucide valide, afficher `Award` par défaut

Toutes les données sont chargées côté serveur dans un Server Component — pas d'appel API client pour cette page.

---

## KPI de validation

- `npx prisma db seed` → 7 badges, vérifier `SELECT icon FROM "Badge"` → noms Lucide valides
- Terminer un cours → cron → `SELECT "totalXp" FROM "EleveProgress"` → +10 XP
- `addXpEvent` appelé 2 fois avec le même `(eleveId, type, referenceId)` → XP non doublé (contrainte unique en base)
- Badge `premier-pas` déclenché après 1er cours TERMINE → `SELECT * FROM "EleveBadge"` → 1 ligne, `pendingBadgeSlug = 'premier-pas'`
- Charger le dashboard élève → modale affichée avec icône Lucide (inspecter le DOM — zéro emoji)
- Fermer la modale → `pendingBadgeSlug = null` en base → rechargement → modale ne réapparaît pas
- Rendre un devoir en avance (>1h avant deadline) → XP `DEVOIR_RENDU` (+15) + `DEVOIR_RENDU_EN_AVANCE` (+5) = total +20
- Page "Mon suivi" : barre de progression cohérente avec `totalXp` en base
- `EleveProgress.totalXp` == somme de tous les `XpEvent.xpGained` de l'élève (invariant absolu)
- `grep -rn "console\.log" ./app ./lib` → 0 résultat
- `tsc --noEmit` et `eslint` → 0 erreur

---

## Rappels architecturaux

- `/lib/gamification.ts` est un module serveur pur — pas d'import de hooks React, pas de `'use client'`
- Les icônes Lucide sont résolues dynamiquement depuis le nom stocké en base : `(LucideIcons as Record<string, ComponentType>)[badge.icon]` — gérer le cas où le nom est invalide
- L'idempotence XP est garantie par la contrainte SQL `@@unique([eleveId, type, referenceId])` — pas par du code applicatif. Toute tentative d'insert dupliqué lève P2002 → traité comme no-op.
