**Stack validée — Plateforme de cours particuliers v1**

---

**Objectif**

Une plateforme web permettant à des professeurs particuliers de gérer leurs cours, leur agenda et leurs paiements. Les parents réservent et paient en avance sur la plateforme ; le professeur reçoit ses paiements directement sur son compte Stripe Connect au moment de la réservation (mode *destination charges*, sans escrow applicatif côté plateforme). La plateforme prélève une commission automatique sur chaque transaction. L'élève accède à un espace de cours (leçons, devoirs) uniquement s'il a un cours à venir.

---

**Stack**

| Couche | Technologie | Version |
|---|---|---|
| Framework | Next.js | 15 (App Router) |
| Langage | TypeScript | 5.x |
| Runtime | Node.js | 22 LTS |
| Base de données | PostgreSQL via Neon | 16 |
| ORM | Prisma | 5.x |
| Auth | Auth.js | v5 |
| Paiements | Stripe Connect | latest |
| Agenda | Google Calendar API | v3 |
| Emails | Resend + React Email | latest |
| Stockage fichiers | Cloudflare R2 | — |
| Déploiement | Vercel | — |
