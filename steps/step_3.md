# STEP 3 — Dashboard administrateur

**Objectif :** un ADMIN peut lister les utilisateurs, les bannir/débannir, consulter les logs de connexion (sans IP) et visualiser les transactions.
**Dépend de :** step_1.
**Branche :** `feat/admin-dashboard`

---

## Ce que tu livres à la fin de cette étape

- Dashboard admin avec sidebar de navigation
- Gestion des utilisateurs (liste paginée, bannissement)
- Logs de connexion paginés (sans IP, conformité RGPD)
- Transactions paginées avec statuts

---

## Ordre d'exécution

### 1. Layout admin

Crée `/app/(auth)/admin/layout.tsx`.

Vérifie côté serveur que `session.user.role === 'ADMIN'`. Si non → redirect `/login`.
Ce check est en plus du middleware — défense en profondeur.

La sidebar contient : Utilisateurs, Logs, Transactions.

### 2. Gestion des utilisateurs

Crée `GET /api/admin/users`.

Paramètres query : `page` (défaut 1), `search` (username ou email), `role` (filtre optionnel).
Retourne : liste paginée de 20 utilisateurs avec `id`, `email`, `username`, `role`, `bannedAt`, `createdAt`.
Jamais de `passwordHash` dans la réponse — utilise `select` explicite sur tous les champs.

Crée `POST /api/admin/users/[userId]/ban`.

Logique :
- Si `bannedAt = null` → set `bannedAt = now()` (bannir)
- Si `bannedAt != null` → set `bannedAt = null` (débannir)
- Un ADMIN ne peut pas se bannir lui-même
- Retourner le nouvel état

Page `/admin` :
- Tableau paginé des utilisateurs
- Champ de recherche (debounce côté client)
- Bouton Bannir/Débannir par ligne
- Confirmation avant action (modale simple)

### 3. Logs de connexion

Crée `GET /api/admin/logs`.

Paramètres query : `page`, `userId` (filtre optionnel).
Retourne : `id`, `userId`, `username`, `userAgent`, `createdAt`.
Vérifier explicitement dans le `select` Prisma qu'`ipAddress` n'est pas sélectionné — le champ n'existe pas dans le schéma (RGPD), mais documenter clairement l'intention.

Page `/admin/logs` :
- Tableau paginé
- Filtre par utilisateur
- Pas d'action possible — consultation uniquement

### 4. Transactions

Crée `GET /api/admin/transactions`.

Paramètres query : `page`, `status` (filtre optionnel).
Retourne : `id`, `amount`, `platformFee`, `profAmount`, `status`, `createdAt`, `booking.subject`, `booking.prof.username`, `booking.payer.username`.
Tous les montants en centimes dans l'API — la conversion en euros est dans le composant d'affichage.

Page `/admin/transactions` :
- Tableau paginé
- Filtres par statut : `EN_ATTENTE`, `REVERSE`, `REMBOURSE`
- Montants affichés en euros avec 2 décimales

---

## KPI de validation

- Accéder à `/admin` avec un compte PROF → redirect `/login`
- Liste utilisateurs : 20 résultats max par page, pagination fonctionnelle
- Bannir un utilisateur → `bannedAt` non null en base → connexion refusée immédiatement
- Débannir → `bannedAt = null` → connexion possible
- Un ADMIN ne peut pas se bannir lui-même → 403
- Logs : `SELECT * FROM "SessionLog"` → aucune colonne `ipAddress`
- Transactions : montants affichés en euros corrects (3000 centimes → "30,00 €")
- `tsc --noEmit` et `eslint` → 0 erreur

---

## Rappels architecturaux

- Toutes les routes `/api/admin/*` vérifient `role === 'ADMIN'` en première ligne
- `select` explicite sur chaque requête Prisma — jamais de `findMany()` sans `select`
- La pagination se fait côté serveur avec `skip` / `take` — jamais côté client
- Les actions destructives (bannissement) sont des `POST`, pas des `GET`
