# SPEC BDD — Version PostgreSQL

> **Base de données : PostgreSQL 16 via Prisma 5.**
> Plus de MongoDB, plus d'`ObjectId`. Tous les identifiants sont des **UUID (String)**. Les relations sont des **clés étrangères SQL** explicites. Les tableaux de scalaires (ex. `String[]`) sont supportés nativement par Prisma + PostgreSQL.
>
> **RGPD — règles appliquées dans toute la spec :**
> - Pas de collecte d'adresse IP (supprimé de `SessionLog`).
> - Pas de nom/prénom obligatoire : on utilise un `username` (pseudo) librement choisi.
> - Collecte minimale : on ne stocke que ce qui est strictement nécessaire au fonctionnement du service.

---

## 1. Domaines enregistrables

### Utilisateurs
- Professeurs, Parents, Élèves mineurs, Étudiants autonomes, Admins

### Cours / Planning
- Créneaux disponibles du prof (via Google Calendar — pas stockés en base)
- Réservations : date, heure, durée, élève, statut

### Suivi pédagogique
- Devoirs : consigne, statut, rendu, correction
- Questionnaires post-séance prof : émojis d'évaluation, remarques publiques, notes privées
- Questionnaires post-séance élève : émojis de ressenti
- **Suivi gamifié élève : XP, badges, streaks** ← nouveau

### Paiements
- Montant, commission plateforme, montant prof, statut, reversement

### Notifications
- Type, destinataire, statut

---

## 2. Entités et schéma Prisma

### **Table : User**

Toutes les personnes de la plateforme partagent cette table. Le rôle détermine les droits et le dashboard affiché.

```prisma
model User {
  id                  String    @id @default(uuid())
  email               String    @unique
  emailVerified       DateTime?
  username            String    @unique             -- pseudo visible, unique (anti-usurpation)
  image               String?                       -- URL photo (Cloudflare R2), optionnel
  passwordHash        String?                       -- null si connexion OAuth uniquement
  role                Role                          -- ADMIN | PROF | PARENT | ELEVE | ETUDIANT
  bannedAt            DateTime?                     -- null = actif
  onboardingCompleted Boolean   @default(false)     -- utilisé pour les PROF uniquement
  cguAcceptedAt       DateTime?                     -- horodatage acceptation CGU (RGPD : preuve de consentement)
  createdAt           DateTime  @default(now())
  updatedAt           DateTime  @updatedAt
}

enum Role {
  ADMIN
  PROF
  PARENT
  ELEVE
  ETUDIANT
}
```

---

### **Table : ProfProfile**

Informations professionnelles du prof, séparées de `User` pour ne pas polluer la table centrale.

```prisma
model ProfProfile {
  id                   String          @id @default(uuid())
  userId               String          @unique
  user                 User            @relation(fields: [userId], references: [id])
  bio                  String?                        -- max 500 caractères
  hourlyRate           Int                            -- en centimes, jamais de float
  subjects             String[]                       -- ex. ["Maths", "Anglais"]
  levels               String[]                       -- ex. ["Collège", "Lycée"]
  stripeAccountId      String?                        -- Stripe Connect Express ID
  googleCalendarEmail  String?                        -- email du compte Google connecté
  refundPolicy         RefundPolicy                   -- enum
  refundPolicyNote     String?                        -- précisions libres, max 300 chars
  inviteCode           String?         @unique        -- code d'invitation actif (UUID court)
  inviteCodeExpiresAt  DateTime?
}

enum RefundPolicy {
  REFUND_2H
  REFUND_24H
  REFUND_48H
  NO_REFUND
}
```

---

### **Table : ProfAvailability**

Plages hebdomadaires récurrentes du prof. Source de vérité applicative pour le calcul des créneaux libres (Google Calendar n'est utilisé QUE pour créer les événements de cours et générer les liens Meet — pas pour stocker les dispos). Plus simple et plus robuste qu'une convention de titre dans Google Calendar.

```prisma
model ProfAvailability {
  id        String   @id @default(uuid())
  profId    String
  prof      User     @relation("ProfAvailabilityProf", fields: [profId], references: [id])
  dayOfWeek Int                                  -- 0 = dimanche, 6 = samedi
  startHour Int                                  -- heure locale du prof (0-23), minutes implicitement 0
  endHour   Int                                  -- heure locale du prof (1-24)
  createdAt DateTime @default(now())
}
```

---

### **Table : Account** (Auth.js — tokens OAuth)

Gérée automatiquement par Auth.js. Stocke les tokens Google chiffrés.

```prisma
model Account {
  id                String  @id @default(uuid())
  userId            String
  user              User    @relation(fields: [userId], references: [id], onDelete: Cascade)
  type              String                        -- "oauth" ou "credentials"
  provider          String                        -- "google"
  providerAccountId String
  access_token      String?                       -- chiffré par Auth.js
  refresh_token     String?                       -- chiffré par Auth.js
  expires_at        Int?
  scope             String?
  @@unique([provider, providerAccountId])
}
```

---

### **Table : SessionLog**

Historique des connexions. **Pas d'adresse IP stockée (RGPD).** Seul le user-agent navigateur est conservé.

```prisma
model SessionLog {
  id        String   @id @default(uuid())
  userId    String
  user      User     @relation(fields: [userId], references: [id])
  userAgent String?                               -- navigateur uniquement, pas d'IP
  createdAt DateTime @default(now())
}
```

---

### **Table : ProfParent** (relation prof ↔ parent/étudiant)

Table de jonction N-M. Un prof peut avoir plusieurs parents clients, un parent peut avoir plusieurs profs.

```prisma
model ProfParent {
  id       String   @id @default(uuid())
  profId   String
  prof     User     @relation("ProfParentProf", fields: [profId], references: [id])
  parentId String
  parent   User     @relation("ProfParentParent", fields: [parentId], references: [id])
  linkedAt DateTime @default(now())
  @@unique([profId, parentId])
}
```

---

### **Table : ParentEleve** (relation parent ↔ élève)

```prisma
model ParentEleve {
  id                 String   @id @default(uuid())
  parentId           String
  parent             User     @relation("ParentEleveParent", fields: [parentId], references: [id])
  eleveId            String
  eleve              User     @relation("ParentEleveEleve", fields: [eleveId], references: [id])
  parentalConsentAt  DateTime @default(now())          -- consentement parental (RGPD-Kids) à la création de l'enfant
  linkedAt           DateTime @default(now())
  @@unique([parentId, eleveId])
}
```

---

### **Table : Booking**

Un booking = un cours réservé et payé. C'est l'entité centrale de la plateforme.

```prisma
model Booking {
  id             String        @id @default(uuid())
  profId         String
  prof           User          @relation("BookingProf", fields: [profId], references: [id])
  eleveId        String
  eleve          User          @relation("BookingEleve", fields: [eleveId], references: [id])
  payerId        String                               -- PARENT ou ETUDIANT qui a payé
  payer          User          @relation("BookingPayer", fields: [payerId], references: [id])
  startTime      DateTime                             -- UTC obligatoire
  endTime        DateTime                             -- UTC obligatoire
  status         BookingStatus @default(CONFIRME)
  meetLink       String?                              -- lien Google Meet généré
  googleEventId  String?                              -- ID événement Google Calendar
  subject        String                               -- matière du cours
  createdAt      DateTime      @default(now())

  payment        Payment?
  postSeanceProf PostSeanceProf?
  postSeanceEleve PostSeanceEleve?
}

enum BookingStatus {
  CONFIRME
  ANNULE
  TERMINE
}
```

---

### **Table : Payment**

Un paiement = une ligne financière liée à un booking. Relation 1-1 avec Booking.

```prisma
model Payment {
  id                      String        @id @default(uuid())
  bookingId               String        @unique
  booking                 Booking       @relation(fields: [bookingId], references: [id])
  stripePaymentIntentId   String        @unique
  stripeCheckoutSessionId String        @unique
  amount                  Int                          -- total en centimes
  platformFee             Int                          -- commission en centimes
  profAmount              Int                          -- montant net pour le prof en centimes
  status                  PaymentStatus @default(EN_ATTENTE)
  payoutSentAt            DateTime?                    -- null = pas encore reversé au prof
  refundedAt              DateTime?                    -- null = pas remboursé
  createdAt               DateTime      @default(now())
}

enum PaymentStatus {
  EN_ATTENTE
  REVERSE
  REMBOURSE
}
```

---

### **Table : PostSeanceProf**

Questionnaire rempli par le prof 15 min après la fin du cours. Relation 1-1 avec Booking.

```prisma
model PostSeanceProf {
  id               String   @id @default(uuid())
  bookingId        String   @unique
  booking          Booking  @relation(fields: [bookingId], references: [id])
  attendance       Boolean                      -- true = élève présent, false = absent
  attitude         Int                          -- 1 à 5 (icône sélectionnée)
  comprehension    Int                          -- 1 à 5
  confiance        Int                          -- 1 à 5
  needsMeeting     Boolean                      -- true = email envoyé au parent
  remarquesPubliques String?                    -- visible par parent et élève, max 1000 chars
  notesPrivees     String?                      -- visible par le prof uniquement, max 1000 chars
  createdAt        DateTime @default(now())
}
```

---

### **Table : PostSeanceEleve**

Questionnaire optionnel rempli par l'élève après le cours. Relation 1-1 avec Booking.

```prisma
model PostSeanceEleve {
  id               String   @id @default(uuid())
  bookingId        String   @unique
  booking          Booking  @relation(fields: [bookingId], references: [id])
  ressenti         Int                          -- 1 à 5 ("comment s'est passé le cours ?")
  comprehension    Int                          -- 1 à 5 ("est-ce que tu as compris ?")
  confianceMatiere Int                          -- 1 à 5 ("comment tu te sens dans cette matière ?")
  createdAt        DateTime @default(now())
}
```

---

### **Table : Homework**

Devoir assigné par le prof à un élève, rattaché optionnellement à un booking.

```prisma
model Homework {
  id                String         @id @default(uuid())
  profId            String
  prof              User           @relation("HomeworkProf", fields: [profId], references: [id])
  eleveId           String
  eleve             User           @relation("HomeworkEleve", fields: [eleveId], references: [id])
  bookingId         String?
  booking           Booking?       @relation(fields: [bookingId], references: [id])
  title             String
  description       String                       -- Markdown sanitisé
  dueDate           DateTime
  status            HomeworkStatus @default(NON_RENDU)
  renderedContent   String?                      -- réponse écrite de l'élève
  correctionContent String?                      -- correction écrite du prof
  submittedAt       DateTime?                    -- horodatage du rendu (null tant que NON_RENDU)
  createdAt         DateTime       @default(now())

  files             HomeworkFile[]
}

-- Note : il n'existe pas de champ `isSubmittedEarly`. Le bonus XP "rendu en avance" est calculé à la volée
-- côté serveur : `submittedAt < dueDate - 1h`. Pas de duplication d'information.

enum HomeworkStatus {
  NON_RENDU
  RENDU
  CORRIGE
}
```

---

### **Table : HomeworkFile**

Fichiers attachés à un devoir (max 3, 10 Mo chacun, stockés sur Cloudflare R2).

```prisma
model HomeworkFile {
  id         String   @id @default(uuid())
  homeworkId String
  homework   Homework @relation(fields: [homeworkId], references: [id])
  url        String                          -- URL R2
  filename   String
  size       Int                             -- en bytes
  createdAt  DateTime @default(now())
}
```

---

### **Table : ResourceFile**

Fichiers attachés à une ressource partagée par le prof (max 3, 10 Mo chacun, Cloudflare R2). Utilisé uniquement quand `Resource.type = FICHIER`.

```prisma
model ResourceFile {
  id         String   @id @default(uuid())
  resourceId String
  resource   Resource @relation(fields: [resourceId], references: [id])
  url        String                          -- URL R2
  filename   String
  size       Int                             -- en bytes
  createdAt  DateTime @default(now())
}
```

---

### **Table : Resource**

Ressource partagée par le prof à un ou tous ses élèves.

```prisma
model Resource {
  id        String       @id @default(uuid())
  profId    String
  prof      User         @relation("ResourceProf", fields: [profId], references: [id])
  eleveId   String?                             -- null = visible par tous les élèves du prof
  eleve     User?        @relation("ResourceEleve", fields: [eleveId], references: [id])
  title     String
  type      ResourceType
  url       String                              -- URL R2 ou lien externe
  createdAt DateTime     @default(now())
}

enum ResourceType {
  FICHIER
  LIEN
}
```

---

### **Table : EleveProgress** ← NOUVEAU (gamification)

Suivi global de la progression gamifiée de l'élève. Une ligne par élève. Mise à jour à chaque événement gamifié.

```prisma
model EleveProgress {
  id                String   @id @default(uuid())
  eleveId           String   @unique
  eleve             User     @relation(fields: [eleveId], references: [id])
  totalXp           Int      @default(0)        -- XP cumulée depuis le début
  currentStreak     Int      @default(0)        -- nombre de semaines consécutives avec au moins 1 cours
  longestStreak     Int      @default(0)        -- record personnel de streak
  totalCours        Int      @default(0)        -- nombre total de cours effectués
  pendingBadgeSlug  String?                     -- slug du badge à notifier à la prochaine page vue, null si aucun
  updatedAt         DateTime @updatedAt
}
```

---

### **Table : XpEvent** ← NOUVEAU (gamification)

Journal de tous les événements qui ont fait gagner de l'XP à l'élève. Permet de reconstruire l'historique et d'afficher les courbes.

```prisma
model XpEvent {
  id          String      @id @default(uuid())
  eleveId     String
  eleve       User        @relation(fields: [eleveId], references: [id])
  type        XpEventType                       -- source de l'XP
  xpGained    Int                               -- points gagnés lors de cet événement
  referenceId String                            -- clé d'idempotence ex. bookingId, homeworkId, ou "streak-2026-W18"
  bookingId   String?                           -- cours concerné (si applicable)
  homeworkId  String?                           -- devoir concerné (si applicable)
  label       String?                           -- texte affiché dans l'historique
  createdAt   DateTime    @default(now())

  @@unique([eleveId, type, referenceId])        -- garantit qu'un même événement n'attribue jamais l'XP deux fois
}

enum XpEventType {
  COURS_EFFECTUE           -- élève présent au cours (+10 XP de base)
  DEVOIR_RENDU             -- devoir rendu dans les délais (+15 XP)
  DEVOIR_RENDU_EN_AVANCE   -- devoir rendu avant la deadline (+5 XP bonus)
  EVALUATION_PROF_POSITIVE -- attitude ou compréhension >= 4 sur 5 (+10 XP)
  QUESTIONNAIRE_REMPLI     -- élève a rempli son questionnaire post-séance (+5 XP)
  STREAK_SEMAINE           -- cours chaque semaine pendant X semaines (+20 XP par pallier)
}
```

---

### **Table : Badge** ← NOUVEAU (gamification)

Catalogue des badges disponibles sur la plateforme. Défini une fois via `prisma db seed` — **seul seed de toute la plateforme**. Les badges sont des données statiques référencées par `slug` dans le code (`checkAndAwardBadges`). Sans ce seed, la gamification échoue au runtime.

```prisma
model Badge {
  id          String        @id @default(uuid())
  slug        String        @unique             -- identifiant technique ex. "premier-cours"
  label       String                            -- nom affiché ex. "Premier pas !"
  description String                            -- ex. "Tu as terminé ton premier cours"
  icon        String                            -- nom de l'icône Lucide UNIQUEMENT (ex. "Star", "Flame", "Trophy") — pas d'emoji
  xpRequired  Int?                              -- XP seuil pour débloquer (si condition = XP)
  createdAt   DateTime      @default(now())

  eleveBadges EleveBadge[]
}
```

---

### **Table : EleveBadge** ← NOUVEAU (gamification)

Badges obtenus par un élève. Table de jonction Badge ↔ Eleve.

```prisma
model EleveBadge {
  id          String   @id @default(uuid())
  eleveId     String
  eleve       User     @relation(fields: [eleveId], references: [id])
  badgeId     String
  badge       Badge    @relation(fields: [badgeId], references: [id])
  obtainedAt  DateTime @default(now())
  @@unique([eleveId, badgeId])              -- un badge ne peut être obtenu qu'une fois
}
```

---

## 3. Relations entre entités

### User → ProfProfile
1-1. Un seul profil prof par utilisateur de rôle PROF.

### User → User (Parent ↔ Élève)
N-M via `ParentEleve`. Un parent peut avoir plusieurs enfants ; un enfant n'a qu'un seul parent en v1.

### User → User (Prof ↔ Parent/Étudiant)
N-M via `ProfParent`. Un prof a plusieurs clients ; un client peut avoir plusieurs profs.

### Booking (Prof, Élève, Payer)
3 FK vers `User`. Le `payerId` est le PARENT ou ETUDIANT ayant payé. L'`eleveId` est l'enfant qui suit le cours.

### Booking → Payment
1-1. Un booking a toujours un paiement associé (créé par le webhook Stripe).

### Booking → PostSeanceProf / PostSeanceEleve
1-1 chacun. Un questionnaire par acteur par cours.

### ProfAvailability → User (Prof)
N-1. Plusieurs plages hebdomadaires par prof.

### Homework → Booking
N-1 optionnel. Un devoir peut être lié à un booking précis, ou non.

### Resource → ResourceFile
1-N. Une ressource de type FICHIER peut avoir jusqu'à 3 fichiers attachés.

### EleveProgress → User
1-1. Une ligne de progression par élève.

### XpEvent → User (Eleve)
1-N. Tous les événements XP d'un élève.

### EleveBadge → User + Badge
N-M via table de jonction. Un élève peut avoir plusieurs badges, chaque badge peut être obtenu par plusieurs élèves.

---

## 4. Notes de migration MongoDB → PostgreSQL

- Tous les `ObjectId` sont remplacés par `String @id @default(uuid())`.
- Les documents imbriqués (ex. `teacher: { teacher_id, permalink }` dans `courses`) sont remplacés par des **clés étrangères** directes.
- Les tableaux d'objets imbriqués (ex. `students[]` dans `courses`) sont remplacés par des **tables de jonction** (ex. `Booking` avec `eleveId`).
- Les tableaux de scalaires simples (ex. `subjects: String[]`, `levels: String[]`) restent des tableaux PostgreSQL, supportés par Prisma.
- La collection `availabilities` MongoDB est remplacée par la table relationnelle **`ProfAvailability`** (plages hebdomadaires récurrentes). Google Calendar n'est utilisé QUE pour créer les événements de cours et générer les liens Meet — il n'est pas la source de vérité des dispos. Choix de simplicité POC : pas de convention `[DISPO]` fragile dans les titres GCal.
- Les collections `modules` et `submodules` sont **hors scope v1** et ne sont pas créées.
- La collection `reports` MongoDB est remplacée par `PostSeanceProf` (plus structuré, scores numériques 1-5, champ `attendance` explicite, notes privées/publiques séparées).
- La collection `assignments` MongoDB est remplacée par `Homework` (simplifié, sans `submodules` en v1).