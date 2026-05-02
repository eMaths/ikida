# STEP 5 — Flux de paiement Stripe

**Objectif :** un parent/étudiant paie un ou plusieurs créneaux via Stripe Checkout. Le webhook crée les bookings et payments en base, génère les événements Google Calendar avec Meet, envoie les emails. Les crons gèrent le cycle de vie des paiements.
**Dépend de :** step_4.
**Branche :** `feat/payment`

---

## Ce que tu livres à la fin de cette étape

- Création de session Stripe Checkout
- Webhook Stripe (signature vérifiée, idempotent)
- Bookings et Payments créés en base après paiement confirmé
- Événements Google Calendar avec lien Meet
- Emails transactionnels via Resend (confirmation, notification prof)
- Remboursement selon la politique du prof
- 3 crons : passage en TERMINE, reversement 24h, rappels J-1
- Pages succès et annulation

---

## Ordre d'exécution

### 1. Client Resend

Crée `/lib/resend.ts`.

Instance Resend singleton. Expose des fonctions nommées pour chaque email transactionnel :
- `sendBookingConfirmationToParent` (EMAIL-01)
- `sendBookingNotificationToProf` (EMAIL-02)
- `sendCourseReminderEmail` (EMAIL-03)
- `sendPayoutConfirmationToProf` (EMAIL-04)
- `sendRefundConfirmationToParent` (EMAIL-05)
- `sendCancellationToProf` (EMAIL-06)
- `sendChildAccountEmail` (EMAIL-07)
- `sendMeetingRequestToParent` (EMAIL-08)
- `sendHomeworkSubmittedToProf` (EMAIL-09)

Chaque fonction prend un objet de paramètres typés — jamais de paramètres positionnels.
Les templates HTML sont minimalistes (texte structuré, pas de design complexe).
Jamais d'email envoyé vers une adresse `@ikida.internal`.

### 2. Création Stripe Checkout

Crée `POST /api/bookings/create`.

Ordre des opérations (respecter cet ordre — chaque étape peut retourner une erreur) :
1. Session : PARENT ou ETUDIANT
2. Rate limiting : 10 requêtes / 1 min par userId
3. Validation Zod du body : `profId`, `eleveId`, `subject`, `slots[]`
4. Vérifier que le parent est bien lié au prof (`ProfParent`)
5. Vérifier que l'élève appartient au parent (ou est l'utilisateur lui-même si ETUDIANT)
6. Vérifier que `ProfProfile.stripeAccountId` existe
7. Pour chaque créneau : vérifier qu'aucun booking `CONFIRME` n'existe déjà sur ce créneau (anti-doublon)
8. Calculer le montant total en centimes : durée en fractions d'heure × `hourlyRate`, arrondi avec `Math.round`
9. Calculer la commission : `Math.round(total × PLATFORM_FEE_PERCENT / 100)` — uniquement des divisions entières
10. Créer la session Stripe Checkout avec `payment_intent_data.transfer_data.destination` et `application_fee_amount`
11. Stocker les metadata : `payerId`, `profId`, `eleveId`, `subject`, `slots` (JSON stringifié)
12. Retourner `{ checkoutUrl }`

### 3. Webhook Stripe

Crée `POST /api/webhooks/stripe`.

Règles absolues :
- Lire le body en `Buffer` brut avec `req.arrayBuffer()` **avant** tout autre traitement
- Vérifier la signature avec `stripe.webhooks.constructEvent()` — signature invalide → 400 immédiat, rien d'autre
- Idempotence : vérifier qu'un `Payment` avec le même `stripePaymentIntentId` n'existe pas déjà avant de créer

Traitement de `checkout.session.completed` :
1. Extraire et parser les metadata
2. Vérifier l'idempotence
3. Pour chaque créneau : créer un `Booking` (status `CONFIRME`), créer l'événement Google Calendar, récupérer le `meetLink`
4. Créer un `Payment` par booking avec les montants calculés depuis `amount_total`
5. Envoyer EMAIL-01 au parent et EMAIL-02 au prof (erreur d'email non bloquante — logguer et continuer)
6. Retourner `{ received: true }` dans tous les cas (Stripe doit recevoir 200)

### 4. Pages succès et annulation

`/booking/[profId]/success` : confirmation visuelle, lien vers le dashboard.
`/booking/[profId]/cancel` : message neutre, bouton réessayer.

Ces deux pages vérifient la session (route protégée).

### 5. Remboursement

Crée `POST /api/bookings/[bookingId]/refund`.

Ordre :
1. Session : PARENT ou ETUDIANT
2. Vérifier que `booking.payerId === session.user.id`
3. Vérifier `booking.status === 'CONFIRME'`
4. Vérifier `payment.status === 'EN_ATTENTE'` — si `REVERSE`, le remboursement est impossible
5. Appliquer la politique du prof :
   - `REFUND_2H` : refus si cours dans moins de 2h
   - `REFUND_24H` : refus si cours dans moins de 24h
   - `REFUND_48H` : refus si cours dans moins de 48h
   - `NO_REFUND` : refus systématique → 403
6. `stripe.refunds.create({ payment_intent: ... })`
7. Transaction Prisma : `Booking.status = 'ANNULE'`, `Payment.status = 'REMBOURSE'`, `Payment.refundedAt = now()`
8. Envoyer EMAIL-05 au parent et EMAIL-06 au prof

### 6. Crons

Crée `vercel.json` à la racine avec les 3 crons.

**Cron 1** — `GET /api/cron/complete-bookings` (toutes les 15 min) :
Passe en `TERMINE` les bookings `CONFIRME` dont `endTime + 15 min` est dépassé.
Après le `updateMany`, récupérer les bookings mis à jour et appeler `addXpEvent` + `checkAndAwardBadges` pour chaque élève concerné (les fonctions de gamification ne sont pas encore créées — les appeler dans step_8, ajouter un TODO ici).

**Cron 2** — `GET /api/cron/payouts` (toutes les heures) :
En mode *destination charges*, Stripe a déjà routé l'argent au moment du Checkout. Ce cron NE fait PAS de `stripe.transfers.create` — il bascule juste un statut applicatif.
Trouve les `Payment` avec `status = 'EN_ATTENTE'` dont le `booking.endTime` est dépassé depuis plus de 24h.
Met à jour `Payment.status = 'REVERSE'` et `payoutSentAt = now()`.
Envoie EMAIL-04 au prof.
Traiter par lots de 100 max.

**Cron 3** — `GET /api/cron/reminders` (tous les jours à 8h UTC) :
Trouve les bookings `CONFIRME` dont `startTime` est entre demain 00h00 et demain 23h59 UTC.
Envoie EMAIL-03 au prof et au payeur (jamais vers `@ikida.internal`).

Chaque cron vérifie `Authorization: Bearer {CRON_SECRET}` en première ligne — absent ou incorrect → 401 immédiat.

---

## KPI de validation

- Sélectionner un créneau → cliquer paiement → page Stripe Checkout s'ouvre avec le bon montant
- Payer avec la carte de test `4242 4242 4242 4242` → redirect vers `/success`
- `SELECT * FROM "Booking"` → 1 ligne avec `status = 'CONFIRME'` et `meetLink` non null
- `SELECT amount, "platformFee", "profAmount" FROM "Payment"` → 3 entiers, pas de float, `amount = platformFee + profAmount`
- EMAIL-01 et EMAIL-02 reçus (vérifier Resend dashboard)
- Webhook avec signature invalide → 400
- Appeler le webhook deux fois avec le même `payment_intent_id` → 1 seul `Payment` en base
- Cron sans `Authorization` → 401
- Remboursement valide → `Payment.status = 'REMBOURSE'`, EMAIL-05 reçu
- Remboursement sur cours `NO_REFUND` → 403
- Remboursement sur `Payment.status = 'REVERSE'` → 409
- `tsc --noEmit` et `eslint` → 0 erreur

---

## Rappels architecturaux

- Le webhook doit retourner 200 à Stripe même en cas d'erreur interne (email raté, Calendar raté) — logguer et continuer
- Les montants : **toujours** des entiers. Aucun float dans la base, aucun float dans les calculs
- En *destination charges*, `transfer_data.destination` route l'argent immédiatement vers le compte Connect du prof. Le cron `payouts` ne fait QUE marquer la base — jamais d'appel Stripe Transfer.
- Jamais de création de Booking en dehors du webhook
