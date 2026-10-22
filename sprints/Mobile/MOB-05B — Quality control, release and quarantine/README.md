# MOB-05B — Contrôle qualité, libération et quarantaine

> **Statut : Terminé** (vérifié le 2026-09-01). Le wizard complet
> (échantillon → aspect → quantités → décision → confirmation), la
> contrainte "un seul contrôle final par lot" appliquée par transaction
> (`qualitySync.ts`, via `runTransaction`), et les brouillons hors ligne
> sont tous réels. Couvert par `qualityValidation.test.ts`,
> `qualityDraft.test.ts`, `QualityControlWizardScreen.test.tsx`, etc.
> **Mise à jour (2026-09-01, MOB-11) :** la résolution de quarantaine
> (MOB-05C), exclue du périmètre initial de ce sprint, a depuis été
> livrée — voir le dossier `MOB-11 — MVP completion (...)`. Un lot en
> quarantaine peut désormais être libéré ou rejeté via
> `/quality-control/[id]/resolve`, sans jamais modifier l'enregistrement
> de quarantaine d'origine.

## Objectif

Terminer le parcours **Mode Production & stock** après MOB-05A : sélectionner un lot produit, effectuer un contrôle simple, saisir les quantités conformes/rejetées, décider de libérer, mettre en quarantaine ou rejeter le lot, puis enregistrer et synchroniser le résultat sans perte de données.

## Écrans du sprint

1. Lots à contrôler
2. Choisir l’échantillon
3. Vérifier l’aspect
4. Saisir les résultats
5. Décider du lot
6. Vérifier et confirmer + modal de confirmation
7. Contrôle enregistré
8. Liste et détail du lot mis à jour
9. Brouillon, hors-ligne, échec de synchronisation et reprise

## Règles UX obligatoires

- Afficher **Mode Production & stock** sur chaque écran.
- Conserver le header supérieur partagé et le menu inférieur `Accueil · Tâches · + · Activité · Plus` sur toutes les routes normales, y compris liste, détail, succès et reprise de brouillon.
- Les étapes du wizard gardent le bouton retour, le rôle, la progression et le menu inférieur.
- Français simple, grands boutons, pictogrammes explicites, aucune table dense.
- Vert = libéré, jaune = quarantaine, rouge uniquement = rejet/action critique.
- Toujours expliquer la conséquence sur le stock avant confirmation.

## Contraintes métier

- Inspecter le schéma réel mobile, web, règles Firestore et moteur ERP avant toute modification.
- Ne pas inventer pH, Brix, température, seuils laboratoire ou rôle `responsable qualité`.
- Si aucun modèle qualité n’existe, créer le plus petit contrat partagé nécessaire, documenté et validé par règles Firestore.
- Un seul contrôle final actif par lot, sauf si le modèle existant autorise explicitement un nouveau contrôle après quarantaine.
- `quantité conforme + quantité rejetée = quantité contrôlée`.
- La décision doit être cohérente avec les quantités et contrôles.
- Quarantaine/rejet exige un motif; libération ne doit jamais rendre disponible une quantité rejetée.
- Toute conséquence de stock doit être atomique/idempotente et validée côté transaction ou backend autoritaire.
- Aucun double mouvement de stock lors d’un retry ou d’une synchronisation répétée.

## Offline et brouillons

- Autosave local par `uid + rôle + productionId`.
- Accès aux brouillons depuis la liste des lots.
- Création offline avec ID local stable et file de mutations.
- États `brouillon`, `en attente`, `synchronisation`, `synchronisé`, `échec`.
- Reprise après fermeture de l’application.
- Vérification serveur au moment de la synchronisation; conflit explicite si le lot a déjà été contrôlé/modifié.
- Ne jamais supprimer silencieusement un brouillon ou résultat local.

## Acceptation

- Le flux complet fonctionne online et offline.
- Les contrôles invalides et totaux incohérents sont bloqués avec un message simple.
- La confirmation expose clairement la décision et son effet stock.
- Un retry ne crée ni second contrôle ni second mouvement.
- Les permissions sont appliquées dans l’UI et dans Firestore.
- Header, rôle et bottom nav sont présents sur chaque écran applicable.
- Les tests MOB-04 et MOB-05A restent verts.
- Typecheck, lint, tests, règles Firebase et export web passent.

## Hors périmètre

- Analyses laboratoire avancées
- Paramétrage des seuils qualité
- Destruction physique avec approbation administrateur
- Factures, paiements ou intégration Mombongo
- Refonte du shell ou de l’authentification

## Référence

Voir `references/MOB-05B-quality-control.png`. Le texte du présent document prévaut sur les éventuelles erreurs typographiques de l’image générée.
