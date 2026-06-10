# Backlog heavy-hub

## Contexte du projet

Le projet représente un portail membre avec home connectée, bibliothèque, détail contenu, dashboard, notifications et profil.

---

## User story 1 — Réduire les appels réseau redondants (polling des notifications)

- **Contexte** : En tant que membre connecté, je veux que le fil de messages ne se rafraîchisse pas en continu toutes les 7 secondes, afin de limiter le trafic réseau et la charge serveur lorsqu'aucune nouvelle notification n'arrive.
- **Constat technique** : Le composant `NotificationsPage` déclenche `window.setInterval(refresh, 7000)`, qui rappelle l'endpoint `/api/notifications` toutes les 7 s tant que l'écran reste ouvert, sans condition de changement et sans pause lorsque l'onglet passe en arrière-plan. Sur une session type de 5 minutes (l'unité fonctionnelle retenue), cela représente environ 43 requêtes pour un contenu qui ne bouge quasiment pas. À l'échelle de la base utilisateurs, ce trafic est multiplié mécaniquement.
- **Objectif** : Diviser par 8 au minimum le nombre d'appels `/api/notifications` sur une session de 5 min (passer d'environ 43 à 5 ou moins), sans dégrader la fraîcheur perçue des messages.
- **Bonnes pratiques d'éco-conception ciblées** : suppression du polling agressif ; réduction des appels API redondants ; limitation des sollicitations pendant la phase d'usage.
- **Pistes d'implémentation** :
  - Supprimer l'intervalle fixe de 7 s.
  - Rafraîchir à la demande (bouton « Actualiser ») et/ou avec un intervalle ≥ 60 s.
  - Suspendre tout rafraîchissement quand `document.hidden` est vrai (écoute de `visibilitychange`), reprendre au retour sur l'onglet.
  - Optionnel : requêtes conditionnelles (`ETag` / `If-None-Match`) pour obtenir une réponse `304 Not Modified` sans renvoyer le corps.
- **Critères d'acceptation** :
  - Plus aucun appel automatique toutes les 7 s.
  - Le rafraîchissement se fait par action utilisateur ou à intervalle ≥ 60 s.
  - Le polling s'arrête lorsque l'onglet n'est pas actif et reprend au retour.
  - Sur une session de 5 min, le nombre d'appels à `/api/notifications` est ≤ 5.
- **KPI associés** : nombre de requêtes `/api/notifications` par minute et sur 5 min (onglet Réseau du navigateur), baseline vs cible.
- **Écrans / fichiers concernés** : `frontend/src/HubApp.tsx` → `NotificationsPage` (`window.setInterval(refresh, 7000)`) et fonction `refreshNotifications` ; `backend/src/index.ts` → endpoint `/api/notifications`.
- **Gain attendu** : réduction directe du trafic réseau et de la charge serveur, levier identifié comme prioritaire dans l'ACV (« réduire le trafic réseau »).
- **Effort / Priorité** : effort faible — priorité haute (Impact fort / Effort faible).

---

## User story 2 — Réduire le poids des contenus (médias optimisés et différés)

- **Contexte** : En tant que visiteur du portail, je veux que les visuels des contenus soient compressés et chargés uniquement lorsqu'ils deviennent visibles, afin de réduire le poids des pages et le temps d'affichage, en particulier sur mobile et réseau contraint.
- **Constat technique** : Le composant `ContentCard` affiche `<img src={item.heroAsset} alt={item.title} />` sans attribut `loading`, sans dimensions explicites et sans variantes responsives. La même image « héro » est rendue immédiatement (eager) dans la sélection de la home, dans toute la grille de la bibliothèque, dans les contenus liés et dans le hero de la fiche détail. L'avatar du profil (`<img src={profile.avatar}>`) n'est pas optimisé non plus. Toutes les images hors écran sont donc téléchargées dès le chargement.
- **Objectif** : Réduire d'au moins 50 % le poids média de la home et de la bibliothèque, améliorer le LCP et supprimer le décalage de mise en page (CLS proche de 0).
- **Bonnes pratiques d'éco-conception ciblées** : réduction du poids des contenus ; lazy loading des médias ; favoriser des contenus légers.
- **Pistes d'implémentation** :
  - Compresser et redimensionner les assets ; servir des formats modernes (WebP / AVIF).
  - Fournir des variantes responsives (`srcset` / `sizes`) adaptées à la taille d'affichage réelle (carte, vignette, hero).
  - Ajouter `loading="lazy"` et `decoding="async"` sur les images hors du premier écran (cartes au-delà de la première rangée, contenus liés, grille bibliothèque).
  - Renseigner `width` / `height` (ou un ratio) pour éviter le reflow.
  - Optimiser l'avatar (taille et format adaptés à son affichage réel).
- **Critères d'acceptation** :
  - Les images de contenus et l'avatar sont compressés et dimensionnés.
  - `loading="lazy"` appliqué aux visuels hors du premier écran.
  - Variantes responsives en place (`srcset` / `sizes`) ou assets correctement dimensionnés.
  - `width` / `height` renseignés, CLS proche de 0.
  - Poids média de la home et de la bibliothèque réduit d'au moins 50 % et score EcoIndex en hausse.
- **KPI associés** : poids média (Ko) de la home et de la bibliothèque, LCP et CLS (Lighthouse), score EcoIndex de la home — baseline vs cible.
- **Écrans / fichiers concernés** : `frontend/src/HubApp.tsx` → composant `ContentCard`, bloc `hub-detail-hero` (`<img src={heroAsset}>`) et `ProfilePage` (avatar) ; dossier `assets`.
- **Gain attendu** : baisse du volume de données transférées et du temps de rendu côté terminal, levier classique de sobriété sur le poids de page.
- **Effort / Priorité** : effort moyen — priorité haute.

---

## User story 3 — Optimiser le caching (cache HTTP des assets et des réponses)

- **Contexte** : En tant qu'exploitant soucieux de la sobriété, je veux activer un cache navigateur et CDN sur les médias et les données stables, afin d'éviter de retélécharger les mêmes ressources à chaque visite ou navigation.
- **Constat technique** : Le backend désactive tout cache. Le middleware `app.use('/assets', express.static(..., { maxAge: 0 }))` empêche la mise en cache des assets, et un middleware global pose `Cache-Control: no-store` sur l'ensemble des réponses. En conséquence, chaque rechargement ou navigation retélécharge intégralement médias et données, et aucun CDN ne peut les mettre en cache, alors que la majorité des contenus du portail sont stables.
- **Objectif** : Servir les assets et les réponses « cacheables » depuis le cache dès la seconde visite et réduire fortement le poids transféré au rechargement.
- **Bonnes pratiques d'éco-conception ciblées** : optimisation du caching (navigateur + CDN) ; réduction des transferts réseau inutiles.
- **Pistes d'implémentation** :
  - Retirer le `Cache-Control: no-store` global ; ne le conserver que pour les endpoints réellement dynamiques.
  - Définir un `Cache-Control: max-age` long sur `/assets` (idéalement avec empreinte / hash dans le nom de fichier pour gérer l'invalidation).
  - Appliquer un cache court ou `stale-while-revalidate` sur les endpoints semi-statiques (`/api/home`, `/api/library`).
  - Optionnel : `ETag` / `Last-Modified` pour des réponses `304` sur les données peu changeantes.
- **Critères d'acceptation** :
  - Les réponses `/assets` exposent un `Cache-Control` avec une durée adaptée.
  - Le `no-store` global est retiré des réponses cacheables.
  - À la seconde visite / au rechargement, les assets sont servis depuis le cache (disque ou `304`).
  - Le poids transféré au rechargement diminue nettement.
- **KPI associés** : part des requêtes servies depuis le cache à la 2ᵉ visite et poids total transféré au rechargement (onglet Réseau) — baseline vs cible.
- **Écrans / fichiers concernés** : `backend/src/index.ts` → middleware `/assets` (`maxAge: 0`) et middleware global `Cache-Control: no-store`.
- **Gain attendu** : suppression des re-téléchargements inutiles, réduction du trafic réseau et de la charge serveur sur les visites récurrentes.
- **Effort / Priorité** : effort faible — priorité haute.
