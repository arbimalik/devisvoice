# DevisVoice — Instructions projet pour Claude Code

## Contexte général
Application SPA pour artisans français : dictée vocale → devis PDF → signature électronique.
- **Frontend** : HTML/CSS/JS vanilla, SPA mono-fichier (`index.html` ~6000 lignes)
- **Hébergement frontend** : GitHub Pages — CNAME `devisvoice.fr` → `arbimalik.github.io/devisvoice`
- **Backend** : Node.js/Express sur Railway → `devisvoice-backend-production-9ed2.up.railway.app`
- **Auth** : JWT 365 jours dans `localStorage` (`dv_token`), helper `authFetch()` injecte le header
- **Email** : Resend API, expéditeur `devis@devisvoice.fr`, helper `sendEmail()` centralisé
- **IA** : Proxy `/api/claude` → API Anthropic (`claude-sonnet-4-6`)
- **Paiement** : Stripe Subscriptions (plans Starter, Pro) avec 30 jours d'essai gratuit
- **Mobile** : Capacitor (`capacitor.config.json`) pour emballage iOS/Android
- **PDF** : jsPDF 2.5.1 + html2canvas 1.4.1 (CDN)

## Architecture des fichiers — Frontend

| Fichier | Rôle |
|---|---|
| `index.html` | App principale — 6 onglets SPA complets (~6000 lignes) |
| `login.html` | Écran de connexion JWT |
| `onboarding-app.html` | Onboarding mobile (flow actuel, remplace les anciens) |
| `onboarding-web.html` | Onboarding desktop (flow actuel, remplace les anciens) |
| `pricing.html` | Page pricing Stripe publique |
| `pricing-success.html` | Page de confirmation post-paiement Stripe |
| `accept.html` | Signature/acceptation devis par lien token UUID (public, côté client) |
| `devis-pdf.html` | Rendu PDF devis isolé |
| `privacy.html` | Politique de confidentialité |
| `CNAME` | `devisvoice.fr` |
| `capacitor.config.json` | Config Capacitor (packaging app mobile) |

## Architecture — Backend

```
devisvoice-backend/
├── server.js              — Point d'entrée, config Express + middlewares
├── db.js                  — Pool PostgreSQL + CREATE TABLE IF NOT EXISTS au boot
├── middleware/auth.js     — requireAuth (JWT), PUBLIC_ROUTES
├── routes/auth.js         — register, login, verify-token, profile, account, export, stats
├── routes/devis.js        — save, get, list, accept, fusion, share (public), send-devis, send-acceptation
├── routes/factures.js     — save, get, list, statut, send-fin-chantier
├── routes/clients.js      — save, get (autocomplete), siret lookup
├── routes/claude.js       — proxy /api/claude + preferences CRUD
├── routes/stripe.js       — checkout, create-checkout, session, webhook, billing-portal
├── routes/vtc.js          — bon-commande save/get/list, send-bon-commande
├── helpers/email.js       — sendEmail() centralisé via Resend
└── helpers/emailBienvenue.js — template email de bienvenue
```

## Base de données — Tables

### `users`
```
id, email (UNIQUE), prenom, nom, entreprise, telephone, mot_de_passe_hash,
famille, metier, metiers (JSONB), document_type, plan,
plaque (VTC), taux_journalier,
stripe_customer_id, stripe_subscription_id, subscription_status,
subscription_current_period_end, created_at, updated_at
```

### `devis`
```
id (PK), data (JSONB), artisan_email, artisan_nom, client_email, libelle,
famille, accepted (BOOL), accepted_by, accepted_at, signature (base64),
statut ('actif'|'fusionné'), fusion_id,
share_token (UUID DEFAULT gen_random_uuid()), created_at
```
`data` est un JSONB libre : `{ lignes, total_ht, total_ttc, client: { nom, email, … }, famille, numero, … }`

### `factures`
```
id (PK), devis_id (FK → devis), artisan_email, client_nom, numero, libelle,
statut ('non_envoyee'|'envoyee'|'en_attente'|'payee'),
lignes (JSONB), created_at, updated_at
```

### `bon_commande`
```
id (PK, format BC-{annee}-{NNNN}), conducteur_email, conducteur_nom, plaque,
passager_nom, passager_email, passager_tel,
date_commande, date_prise_charge, lieu_prise_charge, destination,
distance_km, montant_ttc, pdf_html, created_at
```

### `clients`
```
id (SERIAL PK), artisan_email, nom, email, telephone, siret,
adresse, ville, code_postal, created_at, updated_at
UNIQUE (artisan_email, nom)
```

### `artisan_prefs`
```
artisan_email (PK), style_data (JSONB), updated_at
```
`style_data` : `observations_recentes`, `formulations_types`, `acompte_habituel`,
`conditions_habituelles`, `metiers_frequents`, `updated_at`

## Familles métiers supportées

| Code | Label affiché |
|---|---|
| `btp` | Bâtiment & Travaux (défaut) |
| `evenementiel` | Événementiel & Traiteur |
| `auto` | Auto & Mécanique |
| `services` | Services & Prestataires |
| `creation` | Création & Design |
| `transport` | Transport & Mobilité (contient le métier `vtc`) |

La famille détermine les corps de métier affichés, les labels de l'onglet Chantiers,
la visibilité du sous-onglet Bon de commande (famille `transport` uniquement) et
le bloc VTC dans Mon Profil.

Métiers supprimés définitivement : Isolation, Taxi, Jardinier, Informatique.

## Endpoints API complets

### Routes publiques (sans JWT)
```
POST /api/users/register
POST /api/users/login
POST /api/users/verify-token
POST /api/stripe/webhook
GET  /api/devis/share/:token    — accès client via UUID (lien email)
```

### Utilisateur
```
GET    /api/users/profile
PUT    /api/users/account
DELETE /api/users/account       — supprime compte + toutes les données
GET    /api/users/export        — export RGPD
GET    /api/stats
```

### Devis
```
POST /api/devis/save            — upsert (retourne share_token)
GET  /api/devis                 — liste artisan
GET  /api/devis/:id
POST /api/devis/accept          — marque accepté + signature
POST /api/devis/fusion          — fusionne N devis acceptés
POST /api/send-devis
POST /api/send-acceptation
```

### Factures
```
POST  /api/factures/save
GET   /api/factures
GET   /api/factures/:id
PATCH /api/factures/:id/statut  — valeurs : non_envoyee / envoyee / en_attente / payee
POST  /api/send-fin-chantier    — envoie facture (client + comptable optionnel)
```

### Clients & SIRET
```
POST /api/clients/save
GET  /api/clients               — liste ou recherche ?q=... (max 10 résultats)
GET  /api/siret/:siret          — lookup API gouvernement
```

### Préférences IA
```
GET    /api/preferences
POST   /api/preferences/update
DELETE /api/preferences/reset
```

### Proxy IA & VTC
```
POST /api/claude

POST /api/bon-commande/save
GET  /api/bon-commande
GET  /api/bon-commande/:id
POST /api/send-bon-commande
```

### Stripe
```
POST /api/stripe/checkout
POST /api/stripe/create-checkout
GET  /api/stripe/session/:sessionId
POST /api/stripe/billing-portal
```

## Plans & Stripe

| Plan | Code | Notes |
|---|---|---|
| Gratuit | `gratuit` | Par défaut à l'inscription |
| Starter | `starter` | Price IDs `price_1TNaVs7…` / `price_1TNaWo7…` |
| Pro | `pro` | Price IDs `price_1TNaXu7…` / `price_1TNaXL7…` |

Trial gratuit : 30 jours sur tous les plans payants.
Webhook gère : `checkout.session.completed`, `subscription.updated`, `subscription.deleted`, `invoice.payment_failed`.

## Conventions de code

- Pas de framework — JS vanilla pur, CSS inline ou dans `<style>`
- Variable globale `BACKEND` pour toutes les requêtes API (auto-détecte localhost vs Railway)
- `localStorage` clés : `dv_token`, `user_email`, `user_famille`, `user_metier`, `user_metiers`, `user_plan`, `user_document_type`, `ent-email`, `theme`
- **Tous les fetch vers le backend passent par `authFetch()`** — jamais `fetch(BACKEND+...)` direct
- Le switcher de famille (`#bloc-dev-famille`) est réservé à `malik.arbi@hotmail.com`
- Migrations DB : `ALTER TABLE ... ADD COLUMN IF NOT EXISTS` au boot dans `db.js` — jamais d'outil externe
- SQL brut uniquement — pas d'ORM

## Micro / Dictée vocale — points sensibles

- iOS Safari : ne **jamais** appeler `stream.getTracks().forEach(t => t.stop())` pendant une
  pause — cela empêche la reprise. Libérer le stream uniquement à la déconnexion complète.
- L'orbe d'animation est un canvas — sur mobile, adapter le rayon R à la taille réelle
  du canvas, ne pas clipper avec des valeurs fixes.
- Utilise Web Speech API (`SpeechRecognition` / `webkitSpeechRecognition`), pas MediaRecorder.

## Assistant IA (Vox)

- Modale de chat conversationnel, appelle `/api/claude`
- Feedback 👍/👎 + reset à la fermeture de la modale
- Pas de persistance de l'historique entre sessions (voulu pour l'instant)
- Ne **PAS** le mettre dans l'onglet Paramètres — il a sa propre zone d'accès

## Responsive mobile — règles établies

- Bottom nav fixe : 5 onglets
- Header simplifié (pas de texte long)
- Sur l'écran Devis mobile : bloc dictée vocale **EN PREMIER**, avant le formulaire
- Pricing mobile : scroll horizontal type App Store
- Éviter `order` et les classes CSS `dark` en mobile sur l'écran devis (casse le micro)

## BASE_URL dans index.html

Actuellement hardcodé : `https://arbimalik.github.io/devisvoice`
Utilisé pour les liens `accept.html` et `devis-pdf.html` dans les emails envoyés aux clients.
Le CNAME `devisvoice.fr` fonctionne aussi — les deux URLs sont valides.
Ne pas changer sans tester que les liens emails continuent à fonctionner.

---

## À nettoyer — en attente de validation prod

Une fois que `onboarding-app.html` (mobile) et `onboarding-web.html` (desktop)
sont validés en production et testés **au moins une semaine sans bug remonté**,
supprimer les fichiers suivants (ancien flow fragmenté) :

- `onboarding.html`
- `onboarding-domaine.html`
- `inscription.html`
- `success.html`

**Checklist avant suppression :**
1. Vérifier qu'aucun lien externe ne pointe vers ces pages
   (signets, emails déjà envoyés aux artisans)
2. Vérifier qu'aucune redirection backend ne pointe vers ces URLs
3. `grep -r "onboarding.html\|onboarding-domaine\|inscription.html\|success.html" .`
   doit revenir vide (hors `.git`)

---

## Décisions produit actées

- Pas de framework JS (rester en vanilla pour garder le contrôle total du bundle)
- Un seul fichier `index.html` pour toute l'app (SPA) — pas de split en modules pour l'instant
- L'export de données existe (`/api/users/export`) mais pas encore finalisé côté UX
- Les notifications dans Paramètres sont présentes visuellement mais la logique push/email
  n'est pas encore implémentée côté backend
