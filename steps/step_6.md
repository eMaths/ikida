# STEP 6 — Dashboard élève : cours, devoirs et ressources

**Objectif :** l'élève accède à ses cours, rend ses devoirs et consulte ses ressources. Le prof crée les devoirs et ajoute les ressources. Un élève sans aucun booking n'accède pas au dashboard.
**Dépend de :** step_5.
**Branche :** `feat/eleve-dashboard`

---

## Ce que tu livres à la fin de cette étape

- Dashboard élève complet (cours à venir, historique, devoirs, ressources)
- Gate "aucun cours" : bloque l'accès au dashboard si aucun booking
- Création et rendu de devoirs
- Correction de devoirs par le prof
- Ajout de ressources par le prof
- Pages prof : liste élèves, liste clients, gestion devoirs

---

## Ordre d'exécution

### 1. Gate "aucun cours"

Dans chaque page du dashboard élève (Server Component), vérifier en première ligne :

`prisma.booking.findFirst({ where: { eleveId, status: { in: ['CONFIRME', 'TERMINE'] } } })`

Si null → redirect vers une page `/dashboard/eleve/no-booking` qui explique que le dashboard s'active après le premier cours.

Cette logique est dans chaque page, pas dans le layout — le layout ne gère pas les redirections conditionnelles.

### 2. Dashboard élève — cours à venir

Page `/dashboard/eleve` :
- Bookings `CONFIRME` avec `startTime >= now()`, triés par date ASC
- Pour chaque booking : matière, prof, date/heure, lien Meet
- Lien Meet : affiché mais inactif si le cours commence dans plus de 15 min. Actif dans les 15 min avant le début et jusqu'à la fin du cours.
- Bannière pour les questionnaires post-séance en attente (step_7)

Page `/dashboard/eleve/history` :
- Bookings `TERMINE` ou `ANNULE`, triés par date DESC
- Score moyen affiché si `PostSeanceProf` existe

### 3. Devoirs

Crée `GET /api/eleve/homework` :
- Devoirs de l'élève connecté, triés par `dueDate ASC`
- `select` : `id`, `title`, `description`, `dueDate`, `status`, `correctionContent`, `submittedAt`, `prof.username`, `files`
- Ne jamais inclure `notesPrivees` — ce champ n'existe pas dans `Homework`, mais le principe s'applique à toute la route

Crée `POST /api/eleve/homework/[homeworkId]/submit` :
1. Vérifier que `homework.eleveId === session.user.id`
2. Vérifier `status === 'NON_RENDU'`
3. Vérifier que `dueDate` n'est pas dépassé
4. Mettre à jour `status = 'RENDU'`, `submittedAt = now()` (le bonus "rendu en avance" n'est PAS un champ stocké — il est calculé à la volée côté gamification : `submittedAt < dueDate - 1h`)
5. Envoyer EMAIL-09 au prof
6. Appeler `addXpEvent(eleveId, 'DEVOIR_RENDU', homeworkId)` ; si `submittedAt < dueDate - 1h` appeler aussi `addXpEvent(eleveId, 'DEVOIR_RENDU_EN_AVANCE', homeworkId)`
7. Appeler `checkAndAwardBadges(eleveId)`

Note : les fonctions de gamification sont créées en step_8. Mettre un TODO ici et les brancher en step_8.

**Upload de fichiers de rendu :** réutiliser `POST /api/upload` avec `type=homework`. Côté serveur : types MIME acceptés `application/pdf`, `image/*`, `application/msword`, `application/vnd.openxmlformats-officedocument.wordprocessingml.document`. Taille max 10 Mo.

### 4. Ressources

Crée `GET /api/eleve/resources` :
- Ressources du prof de l'élève (`eleveId = null` ou `eleveId = session.user.id`)
- L'élève doit avoir au moins un booking avec ce prof
- Retourner : `id`, `title`, `type`, `url`, `prof.username`, `files`, `createdAt`

### 5. Côté prof — création de devoirs

Crée `POST /api/prof/homework` :
1. Vérifier que l'élève a au moins un booking `CONFIRME` ou `TERMINE` avec ce prof
2. Créer le `Homework` avec `status = 'NON_RENDU'`
3. Les fichiers joints sont uploadés via `/api/upload` avant cet appel

Crée `POST /api/prof/homework/[homeworkId]/correct` :
1. Vérifier `homework.profId === session.user.id`
2. Vérifier `status === 'RENDU'`
3. Mettre à jour `status = 'CORRIGE'`, `correctionContent`

### 6. Côté prof — ressources

Crée `POST /api/prof/resources` :
- Type `LIEN` : URL requise
- Type `FICHIER` : fichier uploadé via `/api/upload` requis
- `eleveId` optionnel : si null, visible par tous les élèves du prof ; si renseigné, visible uniquement par cet élève

### 7. Pages prof

Page `/dashboard/prof/students` :
- Liste des élèves distincts qui ont au moins 1 booking avec ce prof
- Pour chaque élève : username, nombre de cours, prochain cours
- Clic sur un élève → panneau latéral avec historique des cours et scores (sans `notesPrivees`)

Page `/dashboard/prof/clients` :
- Liste des parents/étudiants liés (`ProfParent`)
- Pour chaque client : username, email, nombre de cours achetés, montant total payé
- Bouton "Générer lien d'invitation" (appelle `POST /api/prof/invite`)

Page `/dashboard/prof/homework` :
- Devoirs groupés par élève, filtrables par statut
- Formulaire de création de devoir
- Section ressources avec bouton ajout

---

## KPI de validation

- Élève sans booking → `/dashboard/eleve` → redirect `no-booking`
- Élève avec booking → dashboard affiché
- Lien Meet → inactif 30 min avant (vérifier dans DevTools que l'attribut `href` est absent ou désactivé)
- Rendre un devoir → `status = 'RENDU'` en base, EMAIL-09 envoyé
- Rendre un devoir déjà rendu → 409
- Rendre un devoir après `dueDate` → 403
- Upload fichier > 10 Mo → 400 côté serveur
- Prof corrige → `status = 'CORRIGE'`, `correctionContent` visible côté élève
- Ressource de type LIEN → URL visible côté élève
- `SELECT * FROM "Homework"` → aucune colonne `notesPrivees` (le champ n'existe pas dans ce modèle)
- `tsc --noEmit` et `eslint` → 0 erreur

---

## Rappels architecturaux

- `notesPrivees` est un champ de `PostSeanceProf` — il ne doit jamais apparaître dans un `select` destiné à un élève ou un parent, même futur
- Chaque page du dashboard élève = Server Component avec gate en première ligne
- L'upload de fichiers se fait toujours en deux temps : d'abord `/api/upload` → obtenir l'URL, puis l'API métier avec l'URL
- Vérifier systématiquement la propriété de la ressource avant modification (homework.eleveId, homework.profId)
