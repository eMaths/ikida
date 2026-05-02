# STEP 10 — Finalisation, sécurité et déploiement

**Objectif :** audit de sécurité complet, rate limiting, build de production propre, déploiement Vercel.
**Dépend de :** steps 0 à 9 — tous les KPI verts.
**Branche :** `feat/hardening`

---

## Ce que tu livres à la fin de cette étape

- Zéro `console.log`, zéro `any`, zéro secret hardcodé
- Rate limiting sur les routes sensibles
- Validation des variables d'environnement au démarrage (fail-fast)
- Headers de sécurité HTTP configurés
- Build de production sans erreur ni warning
- Migrations appliquées en production, seed exécuté
- Application déployée et smoke tests verts

---

## Ordre d'exécution

### 1. Validation des variables d'environnement

Crée `/lib/env.ts`.

Liste toutes les variables obligatoires. Au démarrage (importé dans `/app/layout.tsx`) : si une variable manque → lancer une `Error` avec la liste des variables manquantes. L'app ne démarre pas en production avec une config incomplète.

### 2. Rate limiting

Crée `/lib/rate-limit.ts`.

Utilise `@upstash/ratelimit` + `@upstash/redis` si `UPSTASH_REDIS_REST_URL` est défini.
Sinon → fallback `Map` en mémoire (acceptable pour le dev, documenté clairement comme non-distribué).

Appliquer le rate limiting sur :
- `POST /api/auth/register` : 5 tentatives / 10 min par IP
- `POST /api/bookings/create` : 10 requêtes / 1 min par userId
- `POST /api/join/[code]` : 10 tentatives / 1 min par IP

### 3. Audit `console.log`

```bash
grep -rn "console\.log" ./app ./lib ./components
```
Résultat attendu : zéro. Supprimer ou remplacer par `console.error` si pertinent.

### 4. Audit type `any`

```bash
grep -rn ": any" ./app ./lib ./components
```
Résultat attendu : zéro.

### 5. Audit secrets hardcodés

```bash
grep -rn "sk_live_\|pk_live_\|sk_test_\|whsec_\|re_\|AKIA" ./app ./lib ./components
```
Résultat attendu : zéro. Tous les secrets viennent de `process.env`.

### 6. Headers de sécurité

Dans `next.config.ts`, configurer les headers HTTP pour toutes les routes :
- `X-Frame-Options: DENY`
- `X-Content-Type-Options: nosniff`
- `Referrer-Policy: strict-origin-when-cross-origin`
- `Strict-Transport-Security: max-age=63072000; includeSubDomains; preload`
- `Content-Security-Policy` : ajuster selon les domaines réels (Stripe, Google, R2)

### 7. Build de production

```bash
npm run build
```

Zéro erreur TypeScript. Zéro warning ESLint. Si des warnings apparaissent dans le build → les traiter tous avant de déployer.

Vérifier la taille des bundles (sortie `npm run build`) — signaler à l'humain si une page dépasse 200 KB First Load JS.

### 8. Déploiement

```bash
npx prisma migrate deploy   # applique les migrations en production
npx prisma db seed           # insère les 7 badges
```

Vérifier en production :
```sql
SELECT COUNT(*) FROM "Badge";   -- doit retourner 7
SELECT COUNT(*) FROM "User";    -- cohérent avec les inscriptions de test
```

Merge `feat/hardening` dans `main`. Vercel déploie automatiquement depuis `main`.

Tagger la release :
```bash
git tag v1.0.0
git push origin main --tags
```

### 9. Smoke tests en production

Effectuer dans l'ordre sur l'URL de production :

1. Page d'accueil Ikida accessible → titre "Ikida", slogan "Apprenez mieux, gérez moins"
2. Inscription + connexion PROF → onboarding accessible
3. Headers de sécurité présents : `curl -I https://votre-domaine.vercel.app | grep -E "X-Frame|Strict-Transport|X-Content"`
4. Route admin sans session → 401 ou redirect login
5. Webhook Stripe avec signature invalide → 400 : `curl -X POST -H "stripe-signature: bad" https://votre-domaine.vercel.app/api/webhooks/stripe`
6. Cron sans `CRON_SECRET` → 401 : `curl https://votre-domaine.vercel.app/api/cron/payouts`
7. Réservation complète → paiement → email reçu → booking en base
8. Badge déclenché → modale affichée
9. Dashboard parent → suivi enfant → `notesPrivees` absent du JSON (DevTools Network)

---

## Checklist finale avant de déclarer la v1 terminée

- [ ] `npm run build` → 0 erreur, 0 warning
- [ ] `tsc --noEmit` → 0 erreur
- [ ] `eslint` → 0 erreur, 0 warning
- [ ] `grep console.log` → 0
- [ ] `grep ": any"` → 0
- [ ] `grep secrets hardcodés` → 0
- [ ] `grep notesPrivees` → uniquement dans le périmètre PROF
- [ ] `SELECT * FROM "SessionLog"` → pas de colonne `ipAddress`
- [ ] `SELECT "passwordHash" FROM "User" LIMIT 1` → commence par `$2b$12$`
- [ ] `SELECT amount FROM "Payment" LIMIT 10` → tous des entiers (pas de float)
- [ ] `SELECT COUNT(*) FROM "Badge"` → 7
- [ ] Smoke tests 1 à 9 → tous verts
- [ ] Headers de sécurité présents en production

---

## Features hors scope v1 (ne pas implémenter, ne pas mentionner dans l'UI)

D'après le CDCF — à garder pour v2 :
- Messagerie interne prof-parent
- Avis publics sur les profs
- Moteur de recherche de profs
- Application mobile
- Multi-devises
- Facturation automatique PDF

---

## Rappels architecturaux

- Le déploiement se fait uniquement depuis `main` — jamais depuis une branche feature
- Les migrations de production sont irréversibles — vérifier `prisma migrate status` avant `migrate deploy`
- Le seed est idempotent (`upsert`) — peut être relancé sans risque
