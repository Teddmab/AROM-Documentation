# Prompt Claude — MOB-05B

Implémente **MOB-05B — Contrôle qualité, libération et quarantaine** dans `AROM-Mobile` et, uniquement si nécessaire, le contrat minimal correspondant dans `AROM-Backend` et le miroir de types de `AROM-Production`.

Lis d’abord le `README.md` de ce sprint et utilise l’image jointe `MOB-05B-quality-control.png` comme référence visuelle. Le README est la source fonctionnelle; l’image fixe l’organisation et la hiérarchie visuelle.

## Règles impératives

1. Inspecte avant d’éditer : modèles Production/Stock, repositories, sync queue, drafts MOB-04/MOB-05A, règles Firestore/Storage, transactions, routes, `AppShell`, header et bottom nav.
2. Réutilise exactement l’architecture MOB-05A : liste/détail, wizard, autosave, reprise, mutation offline, idempotence et tests.
3. Ne fabrique aucun champ métier ou endpoint. Cherche d’abord un modèle qualité réel. S’il n’existe pas, propose dans ton compte-rendu initial le contrat minimal requis, puis implémente seulement des données observables : lot, vérifications simples, quantités, décision, motif/commentaire, preuve éventuelle, auteur, dates et statut de sync. Aucun pH/Brix/température sans source existante.
4. Ne modifie pas les formules de rendement de MOB-05A.
5. Ne mélange pas ce sprint avec Administration, finance, Mombongo, facture ou paiement.

## Repères techniques hérités de MOB-05A (à réutiliser tel quel)

Écrit après l'implémentation réelle de MOB-05A dans une session précédente —
ces repères évitent de redécouvrir des contraintes déjà rencontrées et
résolues à ce moment-là.

- **Module de référence** : `AROM-Mobile/src/features/production/`
  (`model.ts`, `productionDraft.ts`, `productionValidation.ts`,
  `productionAvailability.ts`, `productionSync.ts`, `productionRepository.ts`,
  `productionStatus.ts`, `useProductionSync.ts`, `useDraftCount.ts`,
  `components/`) et les écrans `src/app/production-new.tsx`,
  `src/app/productions.tsx`, `src/app/production/[id].tsx`. Copie leur
  structure fichier par fichier plutôt que de la redéduire — c'est le
  patron que le README demande de réutiliser "exactement".

- **Décision d'architecture à prendre en premier** : la décision qualité
  (libérer/quarantaine/rejeter) modifie-t-elle le document `productions/{id}`
  existant, ou crée-t-elle un enregistrement séparé (ex. un nouveau
  `qualityControls/{id}` référençant `productionId`) ? Les deux sont
  défendables, mais ce choix détermine tout le reste (règles Firestore,
  entity de la mutation queue, requête de l'écran liste). À trancher tôt,
  dans le compte-rendu initial, pas en cours de route.
  - Si extension de `productions/{id}` : étendre `isValidProduction()` dans
    `AROM-Backend/firestore.rules` (ajouté en MOB-05A) avec les nouveaux
    champs qualité — ne pas créer une deuxième fonction de validation
    parallèle pour la même collection.
  - `Production.statut` (voir `AROM-Production/src/lib/erp/model.ts`) est une
    chaîne libre côté web (valeurs UI actuelles : "En cours"/"Terminé"), pas
    un enum — aucune valeur "Libéré/Quarantaine/Rejeté" n'existe aujourd'hui
    côté web. Ne pas supposer que le web gère déjà ce concept ; c'est un
    contrat nouveau à documenter, comme le README l'anticipe déjà.

- **`entity` de la mutation queue** : MOB-05A utilise `entity: "production"`
  pour la création de lot. Le contrôle qualité doit utiliser un `entity`
  distinct (ex. `"qualityControl"`) — ne jamais réutiliser `"production"`,
  sinon le filtrage par entity dans `productionRepository.ts`/
  `productionSync.ts` mélange les deux flux dans l'historique et la queue.

- **`operation` de la mutation queue** : `MutationEnvelope.operation` est
  aujourd'hui typé littéralement `"create"` (voir `mutationQueue.ts`). Si le
  contrôle qualité modifie un document existant plutôt que d'en créer un
  nouveau, ce type doit être étendu à `"create" | "update"` de façon
  additive, sans casser les entities existantes — même principe que l'ajout
  additif de `conflict?: boolean` sur `MutationEnvelope` fait en MOB-05A
  pour une raison similaire (éviter de forcer un cas mort dans les switches
  exhaustifs des autres entities).

- **Transaction "un seul contrôle final actif par lot"** : le SDK Firestore
  client ne permet pas de requête live à l'intérieur d'une transaction.
  MOB-05A contourne cette limite dans `productionSync.ts` (fonction
  `syncOne`) en faisant la requête de candidats AVANT `runTransaction`, puis
  en relisant chaque candidat par référence À L'INTÉRIEUR de la transaction
  pour obtenir un instantané cohérent au moment du commit. Réutiliser
  exactement ce patron pour "refuse un contrôle concurrent" plutôt que d'en
  réinventer un ; le risque résiduel documenté dans MOB-05A (une écriture
  concurrente créée entre la requête et le commit reste invisible) s'applique
  ici aussi et doit être documenté de la même façon, pas résolu autrement.

- **États de statut additionnels** : MOB-05A élargit `ProductionStatus`
  (`productionStatus.ts`) avec des états locaux au-delà des `MutationState`
  bruts (ex. "Conflit", "En attente (source)") plutôt que d'ajouter une 5ᵉ
  valeur à `MutationState` partagé. Suivre le même principe pour un éventuel
  état "conflit de contrôle".

- **Catégorie de sync-plan** : si une nouvelle collection est créée, ajouter
  sa propre catégorie dans `syncPlan.ts` sous le rôle `PRODUCTION`, en lisant
  bien le cache de CE rôle (`getCategoryData(uid, "PRODUCTION", ...)`). Un
  bug réel a été corrigé en MOB-05A où une fonction partagée lisait par
  défaut le cache d'un autre rôle (`COLLECTION_AGENT`) au lieu du rôle
  appelant — vérifier ce point pour toute fonction de repository réutilisée
  entre rôles.

## Flux à construire

- `/quality-controls` : lots réellement accessibles au rôle Production & stock, avec état, date, quantité et CTA `Contrôler ce lot`; cartes de brouillons accessibles et reprenables.
- Wizard lié à un `productionId` : échantillon → aspect → quantités → décision → vérification.
- Contrôles d’aspect simples : `Couleur normale`, `Odeur normale`, `Bouteille propre`, avec Oui/Non et photo facultative seulement si Storage/règles peuvent la prendre en charge.
- Quantités : conforme, rejetée et total automatique. Utilise l’unité réelle du lot; pas de conversion implicite.
- Décisions : `Libérer le lot`, `Mettre en quarantaine`, `Rejeter`.
- Motif obligatoire pour quarantaine/rejet. Confirmation renforcée avant une décision restrictive.
- Succès : décision, conséquence stock, état de synchronisation, `Voir le lot`, `Retour aux lots`.

## Intégrité et backend

- Déduis les noms de collections/champs des schémas existants; ne reprends pas littéralement les exemples visuels.
- Ajoute une validation stricte Firestore si une nouvelle collection est requise.
- Autorise uniquement le rôle réel prévu par les conventions actuelles et conserve l’isolation organisationnelle existante.
- À la validation/synchronisation, exécute une opération atomique et idempotente qui relit le lot, refuse un contrôle final concurrent, enregistre le contrôle et applique une seule fois la conséquence stock.
- Une libération rend disponible uniquement la quantité conforme.
- Une quarantaine reste indisponible à la vente.
- Un rejet reste indisponible et ne doit pas être supprimé de l’audit.
- Utilise ID stable/idempotency key selon les patterns MOB-04/MOB-05A.
- En conflit, garde la copie locale et affiche une résolution explicite; aucune écrasement silencieux.

## Offline

- Autosave après chaque changement, clé par utilisateur/rôle/lot.
- Reprise exacte à la dernière étape après relance.
- Queue persistée et retry automatique au retour réseau.
- Échec récupérable avec bouton `Réessayer`.
- Empêche doubles taps et doubles mutations.
- Affiche clairement que la décision est locale tant qu’elle n’est pas synchronisée.

## Design obligatoire

- Utilise le shell partagé existant, pas un second shell.
- Chaque route affiche le header supérieur, le chip `Mode Production & stock` et le bottom nav `Accueil · Tâches · + · Activité · Plus`.
- Dans le wizard, le header contient retour + titre + progression; le bottom nav reste présent sans recouvrir le CTA.
- Safe areas haut et bas obligatoires sur Android/iOS.
- Fond crème, vert AROM, accent jaune; rouge uniquement pour rejet/erreur critique.
- Minimum 48×48 px par cible, texte ≥16 px pour les actions importantes, libellés courts, icône + texte.
- Pas de table dense, jargon technique, champ caché essentiel ou information portée uniquement par une couleur.
- États loading, vide, offline, brouillon, erreur, conflit et succès.
- Les CTA et champs doivent rester visibles avec clavier ouvert et grandes polices.

## Navigation

- Ajoute l’entrée depuis l’accueil Production & stock et l’action rapide autorisée.
- Retour Android cohérent : confirmer avant d’abandonner des changements non sauvegardés; un brouillon autosauvegardé peut être quitté avec message clair.
- Liste, détail et succès conservent le top/bottom menu.
- Ne montre aucune navigation d’application avant authentification; ne touche pas au login.

## Tests obligatoires

- calcul/validation des quantités;
- motifs obligatoires;
- chaque décision et conséquence stock;
- transaction concurrente;
- idempotence et double-submit;
- autosave/reprise;
- offline → pending → retry → synced;
- conflit serveur;
- permissions/règles Firestore;
- présence header + rôle + bottom nav sur chaque route;
- navigation accueil/liste/wizard/détail;
- accessibilité des Oui/Non, décisions et modal;
- non-régression MOB-04/MOB-05A.

Exécute les commandes du dépôt pour typecheck, lint, Jest, tests des règles Firebase et export web. Ne prétends pas avoir réalisé un test appareil s’il n’a pas été fait.

## Réponse finale attendue

Retourne : fichiers modifiés, contrat réel réutilisé/ajouté, comportement stock, offline/idempotence, résultats des tests, vérification header/bottom nav écran par écran, écarts restant par rapport à l’image et test manuel encore requis.
