# DevisVoice — Instructions projet pour Claude Code

## Contexte général
Application SPA (tout-en-un) pour artisans français : dictée vocale → devis PDF.
- **Frontend** : HTML/CSS/JS vanilla, hébergé sur GitHub Pages (CNAME custom)
- **Backend** : Node.js sur Railway → `devisvoice-backend-production-9ed2.up.railway.app`
- **Auth** : JWT stocké en localStorage, guard au chargement de `index.html`
- **Paiement** : Stripe (plans Gratuit, Pro, Expert)
- **Fichier principal** : `index.html` (~368 KB, app SPA complète)

## Architecture des fichiers
| Fichier | Rôle |
|---|---|
| `index.html` | App principale (dashboard, devis, chantiers, facturation, profil, paramètres) |
| `onboarding-app.html` | Onboarding mobile (nouveau flow, remplace les anciens) |
| `onboarding-web.html` | Onboarding desktop (nouveau flow, remplace les anciens) |
| `login.html` | Écran de connexion JWT |
| `pricing.html` | Page pricing Stripe |
| `accept.html` | Acceptation devis par lien token UUID |
| `devis-pdf.html` | Rendu PDF devis isolé |
| `privacy.html` | Politique de confidentialité |

## Familles métiers supportées
`btp`, `evenementiel`, `auto`, `services`, `creation`, `transport`, `vtc`

Métiers supprimés définitivement : Isolation, Taxi, Jardinier, Informatique.

## Conventions de code
- Pas de framework — JS vanilla pur, CSS inline ou dans `<style>`
- Variable globale `BACKEND` pour toutes les requêtes API
- `localStorage` pour : `token`, `ent-email`, `user_famille`, `user_metier`, `user_metiers`
- Tous les fetch vers le backend doivent envoyer `Authorization: Bearer <token>`
- Le switcher de famille dans le bloc dev est réservé à `malik.arbi@hotmail.com`

## Endpoints backend principaux
```
GET/PUT  /api/users/account
GET      /api/users/profile
POST     /api/users/verify-token
GET      /api/stats
POST     /api/devis/save
GET      /api/devis/:id
GET      /api/devis (liste, JWT requis)
POST     /api/send-devis
POST     /api/devis/fusion
GET/POST /api/clients
GET      /api/preferences
POST     /api/preferences/update
DELETE   /api/preferences/reset
POST     /api/bon-commande/save
POST     /api/send-bon-commande
POST     /api/factures/save
POST     /api/send-fin-chantier
POST     /api/claude
GET      /api/siret/:siret
GET      /api/users/export
```

## Micro / Dictée vocale — points sensibles
- iOS Safari : ne **jamais** appeler `stream.getTracks().forEach(t => t.stop())` pendant une
  pause — cela empêche la reprise. Libérer le stream uniquement à la déconnexion complète.
- L'orbe d'animation est un canvas — sur mobile, le clipper au wrap et adapter le rayon R
  à la taille réelle du canvas.

## Assistant IA (Vox)
- Modale de chat conversationnel, appelle `/api/claude`
- Feedback 👍/👎 + reset à la fermeture de la modale
- Pas de persistance de l'historique entre sessions (voulu pour l'instant)
- Ne PAS le mettre dans l'onglet Paramètres — il a sa propre zone d'accès

## Responsive mobile — règles établies
- Bottom nav fixe : 5 onglets
- Header simplifié (pas de texte long)
- Sur l'écran Devis mobile : bloc dictée vocale EN PREMIER, avant le formulaire
- Pricing mobile : scroll horizontal type App Store
- Éviter `order` et les classes CSS `dark` en mobile sur l'écran devis (casse le micro)

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
