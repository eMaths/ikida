# Règles et bonnes pratiques pour l'agent de développement

> Ce fichier définit les règles absolues que l'agent IA doit suivre tout au long du développement de la plateforme.
> Certaines règles sont des interdictions strictes. D'autres sont des obligations positives.
> En cas de doute sur l'une d'elles, l'agent doit demander à l'humain avant d'agir.

---

## 1. Comportement général de l'agent

### Ce que l'agent NE DOIT PAS faire

- **Ne jamais prendre d'initiative non demandée.** Si une décision n'est pas explicitement couverte par le cahier des charges ou ce fichier, l'agent doit s'arrêter et poser une question avant de continuer.
- **Ne jamais répondre à sa propre question.** Si l'agent formule un doute ou une alternative, il doit attendre la réponse de l'humain. Il ne choisit pas lui-même.
- **Ne jamais supposer qu'une exigence est évidente.** Même si quelque chose semble logique, si ce n'est pas écrit, l'agent demande.
- **Ne jamais avancer sur plusieurs features en même temps** sans validation de la feature précédente.
- **Ne jamais marquer une tâche comme terminée** si elle n'est pas testée manuellement ou par un critère d'acceptation vérifié.
- **Ne jamais inventer une solution à un problème bloquant.** Exposer le problème clairement à l'humain avec les options possibles, sans en choisir une.

### Ce que l'agent DOIT faire

- **Partager ses doutes explicitement**, sous forme de question claire et courte, avant chaque choix architectural ou fonctionnel non spécifié.
- **Travailler feature par feature**, dans l'ordre de la checklist du CDCF. Une feature = un commit.
- **Pousser sur Git régulièrement.** Minimum : un commit par feature terminée et validée. Message de commit en français, préfixé : `feat:`, `fix:`, `refactor:`, `chore:`, `docs:`.
- **Documenter au fur et à mesure**, pas à la fin. Chaque fonction utilitaire, chaque API Route, chaque composant complexe reçoit un commentaire JSDoc au moment où il est écrit.
- **Rendre compte à l'humain après chaque étape**, selon le format défini à la section 13 ci-dessous.

---

## 13. Reporting obligatoire vers l'humain

> Cette section est non-négociable. L'agent ne passe jamais à l'étape suivante sans avoir produit ce rapport et reçu un accord explicite de l'humain.

### Quand rendre compte

L'agent produit un rapport à l'humain dans les situations suivantes :

- **Après avoir terminé une feature** (même partielle).
- **Avant de commencer une nouvelle feature.**
- **Dès qu'un problème est rencontré**, quelle que soit sa gravité — blocage, incertitude, comportement inattendu d'une API tierce, ambiguïté dans le CDCF, etc.
- **Avant toute modification du schéma Prisma.**
- **Avant toute installation d'une nouvelle dépendance.**
- **Dès qu'une décision non couverte par le CDCF ou ce fichier se présente.**

### Format du rapport (à utiliser systématiquement)

```
--- RAPPORT AGENT ---

FAIT :
- [liste précise de ce qui a été implémenté et fonctionne]

A FAIRE (prochaine étape) :
- [prochaine feature ou sous-tâche selon la checklist CDCF]

PROBLEMES / BLOCAGES :
- [tout ce qui ne fonctionne pas, tout ce qui est incertain, toute ambiguïté]
- AUCUN si tout va bien — ne pas omettre cette section même vide

QUESTIONS (réponse humain requise avant de continuer) :
- [chaque question, une par ligne, numérotée]
- AUCUNE si aucune question — ne pas omettre cette section même vide

---------------------
```

### Règles de reporting strictes

- **Ne jamais sauter la section "PROBLEMES".** Si aucun problème, écrire explicitement `AUCUN`. Ne pas omettre la section sous prétexte que tout va bien.
- **Ne jamais sauter la section "QUESTIONS".** Même réponse : `AUCUNE` si vide.
- **Ne jamais écrire "c'est fait" sans détailler ce qui est fait.** La section FAIT doit lister chaque fichier créé ou modifié, chaque route implémentée, chaque comportement validé.
- **Ne jamais masquer un problème** en le contournant sans le signaler. Si une API tierce se comporte différemment de la doc, si un type Prisma ne correspond pas, si un comportement Auth.js est inattendu — c'est un problème à déclarer, pas à patcher silencieusement.
- **Ne jamais écrire "mock temporaire" ou "à remplacer plus tard"** dans le code sans en informer l'humain dans la section PROBLEMES du rapport. Et même dans ce cas, l'humain doit valider explicitement avant que le mock soit accepté.
- **Ne jamais présenter un résultat partiel comme un résultat complet.** Si une feature est implémentée à 80%, le rapport indique 80% fait et liste précisément ce qui manque.

---

## 2. Interdictions absolues de code

### Données et sécurité

- **Zéro float pour les montants.** Tout calcul financier se fait en centimes (entiers). `Math.round()` interdit sauf pour la conversion d'affichage uniquement.
- **Zéro donnée bancaire côté serveur.** Aucune carte, IBAN ou donnée Stripe sensible ne transite par les routes Next.js. Stripe est l'unique responsable.
- **Zéro adresse IP stockée.** Ni dans `SessionLog`, ni dans les logs applicatifs, ni ailleurs. Interdit même temporairement.
- **Zéro token OAuth en clair.** Les tokens Google (`access_token`, `refresh_token`) sont gérés et chiffrés exclusivement par Auth.js. L'agent ne les manipule jamais directement hors du callback Auth.js.
- **Zéro secret dans le code source.** Toutes les clés API, secrets et credentials sont dans les variables d'environnement (`.env.local` en dev, variables Vercel en prod). Aucune valeur sensible dans un fichier versionné, même commentée.
- **Zéro `console.log` en production** contenant des données utilisateur, des tokens ou des montants. Les logs de debug sont retirés avant tout commit sur `main`.
- **Zéro endpoint non protégé.** Chaque API Route vérifie le rôle de l'utilisateur via le middleware Auth.js avant toute action. Aucune exception, même pour "tester vite".
- **Zéro validation uniquement côté client.** Toute validation de formulaire doit être dupliquée côté serveur (API Route). Le client peut valider pour l'UX, mais le serveur est l'arbitre.
- **Zéro `any` TypeScript.** Le type `any` est interdit. Si le type est inconnu, utiliser `unknown` et le raffiner explicitement.
- **Zéro SQL brut avec des valeurs non paramétrées.** Toutes les requêtes passent par Prisma. Si un raw query est absolument nécessaire, utiliser `prisma.$queryRaw` avec des tagged template literals uniquement.

### Webhook Stripe

- Le endpoint `/api/webhooks/stripe` doit **toujours** vérifier la signature Stripe avec `stripe.webhooks.constructEvent()` avant de traiter quoi que ce soit. Si la vérification échoue, retourner `400` immédiatement sans aucun traitement.
- Le corps de la requête doit être lu comme un `Buffer` brut (`await req.arrayBuffer()`), pas comme du JSON parsé, pour que la vérification de signature fonctionne.

### Route cron

- Les routes `/api/cron/*` doivent vérifier la présence et la valeur du header `Authorization: Bearer <CRON_SECRET>` avant tout traitement. Si absent ou incorrect, retourner `401` immédiatement.

---

## 3. Interdictions de style et d'interface

- **Zéro emoji dans le code source.** Ni dans les variables, ni dans les commentaires, ni dans les chaînes de caractères côté serveur.
- **Zéro emoji dans l'interface utilisateur.** Toute icône dans le front doit être un composant `lucide-react`. Les emojis sont interdits dans les labels, boutons, titres, messages d'état et notifications. Aucune exception.
- **Zéro style inline (`style={{...}}`)** dans les composants React. Tout le style passe par des classes TailwindCSS.
- **Zéro composant monolithique.** Un composant qui dépasse 150 lignes doit être découpé. Chaque composant a une responsabilité unique.
- **Zéro logique métier dans un composant React.** La logique (calculs, appels API, transformations de données) appartient aux Server Components, aux API Routes ou aux fonctions utilitaires dans `/lib`. Les composants client ne font qu'afficher et émettre des événements.

---

## 4. Interdictions de processus

- **Zéro mock, zéro données simulées.** Aucun `const fakeData = [...]` dans le code livré. Chaque feature est branchée sur la vraie base de données dès son développement.
- **Zéro feature "à moitié faite" committée.** Un commit sur `main` ne contient que du code qui fonctionne de bout en bout selon les critères d'acceptation du CDCF.
- **Zéro `TODO` ou `FIXME` laissé sans issue.** Si un problème est identifié et ne peut pas être résolu immédiatement, l'agent le signale à l'humain avant de continuer. Il ne le commente pas et ne l'ignore pas.
- **Zéro hallucination d'API ou de bibliothèque.** L'agent ne doit utiliser que des APIs et méthodes dont il est certain qu'elles existent dans la version spécifiée. En cas de doute, il demande à l'humain de vérifier la documentation officielle.
- **Zéro duplication de logique.** Si une logique est utilisée à deux endroits, elle est extraite dans une fonction utilitaire dans `/lib`. Principe DRY appliqué systématiquement.

---

## 5. Structure du code et conventions

### Organisation des fichiers

```
/app                    Routes Next.js App Router
  /(public)             Pages publiques (/, /login, /register)
  /(auth)               Pages protégées — layout avec vérification Auth.js
    /dashboard
      /prof/...
      /parent/...
      /eleve/...
      /etudiant/...
    /admin/...
  /api                  API Routes
    /auth/...
    /bookings/...
    /prof/...
    /stripe/...
    /cron/...
    /webhooks/...
/components             Composants React réutilisables
  /ui                   Composants shadcn/ui générés (ne pas modifier manuellement)
  /shared               Composants métier partagés entre rôles
  /prof                 Composants spécifiques au dashboard prof
  /parent               Composants spécifiques au dashboard parent
  /eleve                Composants spécifiques au dashboard élève
/lib                    Logique métier, utilitaires, clients externes
  /prisma.ts            Instance Prisma singleton
  /auth.ts              Configuration Auth.js
  /stripe.ts            Client Stripe
  /google-calendar.ts   Client Google Calendar API
  /resend.ts            Client Resend
  /xp.ts                Logique d'attribution XP et badges (gamification)
  /utils.ts             Fonctions utilitaires générales (formatage, dates...)
/prisma
  /schema.prisma        Schéma Prisma — source de vérité de la BDD
  /seed.ts              Seed des données initiales (badges uniquement)
```

### Nommage

- **Fichiers de composants** : `PascalCase.tsx` — ex. `BookingCalendar.tsx`
- **Fichiers utilitaires et lib** : `kebab-case.ts` — ex. `google-calendar.ts`
- **Variables et fonctions** : `camelCase`
- **Constantes globales** : `SCREAMING_SNAKE_CASE`
- **Types et interfaces TypeScript** : `PascalCase`, préfixés par leur domaine — ex. `BookingWithPayment`, `ProfProfileWithUser`
- **Enums Prisma** : reproduits tels quels dans le code TypeScript, pas re-définis manuellement

### TypeScript

- **Strict mode activé** dans `tsconfig.json`. Ne jamais le désactiver.
- **Pas de `as` forcé** (`as SomeType`) sauf quand absolument nécessaire et commenté.
- **Les types de retour des fonctions** sont explicites pour toutes les fonctions utilitaires dans `/lib`.
- **Les props de composants** sont typées avec une interface nommée, jamais inline anonyme.

---

## 6. API Routes — règles systématiques

Chaque API Route Next.js doit suivre ce patron dans l'ordre :

1. Récupérer la session Auth.js avec `auth()` (Auth.js v5 — ne PAS utiliser `getServerSession`, qui est v4).
2. Vérifier que la session existe — sinon retourner `401`.
3. Vérifier que le rôle de l'utilisateur correspond à l'action — sinon retourner `403`.
4. Valider le corps de la requête avec Zod. Si invalide, retourner `400` avec les messages d'erreur Zod.
5. Exécuter la logique métier.
6. Retourner la réponse.

Toute déviation de cet ordre doit être justifiée par une question à l'humain.

---

## 7. Gestion des erreurs

- **Toute erreur inattendue** dans une API Route est capturée dans un bloc `try/catch` et retourne `500` avec un message générique côté client. Le détail de l'erreur est loggué côté serveur uniquement (sans données sensibles).
- **Les erreurs Prisma** sont distinguées des erreurs génériques. `PrismaClientKnownRequestError` avec code `P2002` (contrainte unique) retourne `409` avec un message explicite.
- **Les erreurs Stripe** sont loggées avec leur `type` et `code`, sans les données de la requête.
- **Côté client**, les erreurs affichent toujours un message utilisateur lisible. Jamais un stack trace, jamais un message d'erreur brut de l'API.

---

## 8. Base de données et Prisma

- **Une seule instance Prisma** dans toute l'application, définie dans `/lib/prisma.ts` avec le pattern singleton pour éviter les connexions multiples en développement.
- **Pas de `prisma.user.findMany()` sans `select` ou `where`** dans les pages qui listent des données. Toujours projeter uniquement les champs nécessaires.
- **Les `notesPrivees` du prof** ne sont jamais incluses dans une requête Prisma destinée au parent ou à l'élève. Vérification systématique des champs retournés.
- **Chaque migration Prisma** est committée avec le code qui en a besoin, dans le même commit.
- **Le fichier `schema.prisma`** est la source de vérité. Si un champ est nécessaire et absent du schéma, l'agent s'arrête, signale le manque à l'humain et attend la validation avant de modifier le schéma.

---

## 9. Sécurité — règles supplémentaires

- **Rate limiting** sur les routes sensibles (`/api/auth/*`, `/api/bookings/create`) : utiliser `@upstash/ratelimit` ou équivalent. Si non spécifié dans le CDCF, demander à l'humain avant d'implémenter.
- **En-têtes de sécurité HTTP** : configurer dans `next.config.ts` les headers `X-Frame-Options`, `X-Content-Type-Options`, `Referrer-Policy`, `Permissions-Policy`. Ne pas exposer `X-Powered-By`.
- **CORS** : les API Routes ne sont accessibles que depuis le même domaine. Pas de configuration CORS permissive.
- **Uploads de fichiers** : vérifier le type MIME côté serveur (pas seulement l'extension), limiter la taille à 10 Mo par fichier, ne stocker que sur Cloudflare R2. Jamais dans le dépôt Git, jamais dans la base de données.
- **Les URLs R2** publiques sont signées ou expirent si les ressources sont privées. Demander à l'humain quelle stratégie adopter avant d'implémenter.

---

## 10. Git et versioning

- **Branche principale** : `main` — code toujours déployable.
- **Développement** : une branche par feature, nommée `feat/nom-de-la-feature` — ex. `feat/booking-flow`, `feat/gamification-xp`.
- **Un commit = une feature terminée et validée.** Pas de commits "work in progress" sur `main`.
- **Messages de commit** : en français, préfixés selon Conventional Commits :
  - `feat:` nouvelle fonctionnalité
  - `fix:` correction de bug
  - `refactor:` restructuration sans changement de comportement
  - `chore:` mise à jour de dépendances, configuration
  - `docs:` documentation uniquement
- **Avant chaque push**, l'agent s'assure que `tsc --noEmit` et `eslint` passent sans erreur.
- **Le fichier `.env.local`** est dans `.gitignore`. Un fichier `.env.example` avec les noms de variables (sans valeurs) est versionné et tenu à jour.

---

## 11. Documentation

- **JSDoc obligatoire** pour toutes les fonctions dans `/lib`. Minimum : une ligne de description, les paramètres typés, le type de retour.
- **Commentaire de section** en tête de chaque API Route, décrivant : méthode HTTP, rôle requis, description en une phrase.
- **Le CDCF (`CDCF_Cours_Particuliers_v1.md`) et la SPEC BDD (`SPEC BDD.md`)** sont mis à jour si une décision de développement modifie le périmètre fonctionnel ou le schéma. L'agent ne modifie pas ces fichiers sans accord explicite de l'humain.
- **`README.md`** tenu à jour avec : prérequis, variables d'environnement requises, commandes de lancement, de migration et de seed.

---

## 12. Checklist avant chaque PR / push sur main

L'agent vérifie chaque point avant de déclarer une feature terminée :

- [ ] `tsc --noEmit` : zéro erreur TypeScript
- [ ] `eslint` : zéro warning, zéro erreur
- [ ] Critères d'acceptation du CDCF pour cette feature : tous vérifiés manuellement
- [ ] Aucun `console.log` de debug restant
- [ ] Aucun `TODO` / `FIXME` non signalé à l'humain
- [ ] Aucune variable d'environnement hardcodée
- [ ] `.env.example` mis à jour si une nouvelle variable a été ajoutée
- [ ] JSDoc rédigé pour toutes les nouvelles fonctions dans `/lib`
- [ ] Migration Prisma committée si le schéma a changé
- [ ] `prisma db seed` fonctionne sans erreur sur une base vide
