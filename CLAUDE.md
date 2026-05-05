## À nettoyer après validation du nouveau onboarding

Une fois que onboarding-app.html (mobile) et onboarding-web.html
(desktop) sont validés en production et testés pendant
au moins une semaine sans bug remonté, supprimer les fichiers
suivants qui appartiennent à l'ancien flow fragmenté :

- onboarding.html
- onboarding-domaine.html
- inscription.html
- success.html

Avant de supprimer :
1. Vérifier qu'aucun lien externe ne pointe vers ces pages
   (signets, emails déjà envoyés aux artisans, etc.)
2. Vérifier qu'aucune redirection backend ne pointe vers
   ces URLs
3. Faire un grep complet du projet pour confirmer que les
   noms de ces fichiers n'apparaissent nulle part
