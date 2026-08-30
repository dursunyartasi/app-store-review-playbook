# Soumission à l'App Store

Le build est envoyé et `VALID`. Viennent maintenant les métadonnées, les déclarations et les notes de revue — c'est là que la plupart de nos refus se sont réellement joués.

## Créer l'enregistrement (une fois)

**L'enregistrement de l'application dans App Store Connect ne peut pas être créé par API.** On a essayé et confirmé ; fais-le dans le navigateur. Le bundle ID s'y lie définitivement.

Tout le reste — attacher le build, métadonnées, classification par âge, notes de revue, soumission — se pilote par l'API ASC.

## Captures d'écran

- **Le type de capture `APP_IPHONE_69` n'existe pas.** Le plus grand type iPhone accepté par l'API est `APP_IPHONE_67` (1290×2796). Les images rendues en 1320×2868 pour l'appareil 6,9" sont **refusées**. Envoie du 6,7" et laisse Apple agrandir.
- `whatsNew` **ne peut pas être modifié sur une première version** — 409, « cannot be edited at this time ». Il n'existe que pour les mises à jour.

## Classification par âge

- Les types de champs sont mélangés : certains BOOLEAN (`messagingAndChat`, `userGeneratedContent`, `advertising`), d'autres énumérations STRING (`contests`, `profanityOrCrudeHumor` → `NONE` / `INFREQUENT_OR_MILD` / `FREQUENT_OR_INTENSE`). Le mauvais type renvoie 409, et le message d'erreur nomme le bon ensemble.
- **Apple a changé les tranches en 2025 : le 12+ n'existe plus.** Ce sont 4+, 9+, 13+, 16+, 18+.
- Des réponses honnêtes peuvent donner 4+ ; remonte avec `ageRatingOverrideV2` (par exemple `THIRTEEN_PLUS`).
- **Si l'application a la moindre dimension « rencontrer des gens / réseautage », déclare `matureOrSuggestiveThemes` au moins en `INFREQUENT_OR_MILD`.** Le laisser à néant nous a valu un refus 2.3.6.

## Déclaration App Privacy

- **Un numéro de pièce d'identité nationale n'est pas une « Sensitive Info ».** La catégorie sensible d'Apple couvre l'origine, la religion, l'orientation sexuelle, la biométrie et assimilés ; une pièce d'identité n'y figure pas, donc la bonne case est **« Other Data Types »**.
- **Les données bancaires que tu stockes toi-même sont « Collected ».** Apple n'exonère que si le prestataire de paiement les détient et que tu n'y as pas accès.
- ⚠️ **Ne clique pas à l'aveugle dans l'assistant.** Il s'affiche à des hauteurs différentes selon le type de donnée, si bien que répéter le même point de clic a produit des réponses comme « L'identifiant utilisateur sert au suivi : OUI » qui étaient simplement fausses. Fais une capture et vérifie l'état final de chaque élément.

## Notes d'App Review — le texte au plus fort effet de levier

L'un de nos refus venait entièrement de ce champ. Le refus 4.2 d'Apple, « small, or niche, set of users », nous a cité notre propre phrase :

> "We are targeting a deliberately **small**, curated early community — **a few dozen** invited members, **not a mass-market** social network."

**Ne présente jamais l'application comme petite, de niche, sur invitation seulement, fermée, privée, destinée à une communauté précise ou non grand public.** Apple lit cela comme de la distribution Ad Hoc, pas comme l'App Store.

Écris plutôt : ouverte à tous, gratuite, téléchargeable de partout ; toute couche curée ou d'adhésion est *optionnelle*. Puis décris, en étapes numérotées, le parcours que la revue peut suivre **sans compte**. Si ce parcours n'existe pas vraiment, construis-le avant de soumettre — c'est exactement ce qui a débloqué notre 4.2.

**Le champ est limité à 4000 caractères.** Le dépasser renvoie 409.

Si l'application vise une cible inhabituelle (Watch, widget, un parcours propre à un appareil), place tout en haut une section « PLEASE READ FIRST » avec les étapes de connexion explicites.

## Compte de démonstration

Coche « Sign-In Required » et fournis les identifiants.

- **Teste-les d'abord sur un appareil.** Un refus 2.1 est venu d'une connexion qui n'avait jamais fonctionné sur le target Watch.
- **Assure-toi que le compte a du contenu.** Dans une application, 16 des 17 événements initialisés étaient dans le passé, la revue aurait ouvert une application vide. Garde un script idempotent qui décale les dates de démo vers l'avant et lance-le avant chaque soumission.
- **Un mur de vérification piégera la revue.** Si une personne inscrite mais non vérifiée ne voit rien, l'application paraît fermée. Laisse les invités parcourir ; n'exige la vérification qu'aux actions d'écriture.
- **Ferme le compte de démo après l'approbation.** Son mot de passe se trouve dans App Store Connect.

## Liens juridiques

Les liens vers les Conditions et la Confidentialité doivent être **cliquables**, s'ouvrir dans un navigateur intégré plutôt que d'éjecter vers Safari, et figurer aussi sur l'écran de **connexion**, pas seulement d'inscription. Du texte brut non cliquable a valu un refus 2.1(a) : la revue n'a pas pu lire les conditions et a refusé sur ce seul motif.

## Si l'application est gratuite mais vend quelque chose quelque part

La 3.1.1 est le piège des applications gratuites et B2B. **Retire tout prix, nom de formule, compteur de crédits, mur payant, bouton d'upgrade et lien d'achat externe.** Un simple nom de formule a suffi à couler un build.

L'argument 3.1.3(f) « Free Stand-alone Apps » **n'a pas suffi à lui seul chez nous.** Le maillon faible était un écran d'inscription public : il se lit comme de la vente en libre-service au consommateur et contredit le « only sold directly by you to organizations » de la 3.1.3(c). On a supprimé l'écran d'inscription et publié avec la seule connexion.

## Soumettre, et resoumettre après un refus

L'ordre compte. Se tromper soumet silencieusement le mauvais binaire.

1. **Deux versions ne peuvent pas être en revue en même temps.** Annule la `reviewSubmission` existante (`canceled=true`) et attends CANCELING → COMPLETE.
2. La version passe en `DEVELOPER_REJECTED` et devient modifiable. Fais un PATCH de la chaîne de version, puis de la relation avec le build.
3. ⚠️ **Le piège de l'échange.** Juste après l'annulation, l'appel d'attachement renvoie 409. Si ton script continue quand même, il soumet l'**ancien** build. Réessaie l'attachement, puis **vérifie** avec `GET /appStoreVersions/{id}/build` avant de soumettre. On a publié le mauvais build de cette façon une fois.
4. ⚠️ `POST reviewSubmissionItems` peut renvoyer 409 `ENTITY_STATE_INVALID` pendant la transition d'état. Ça passe quelques secondes plus tard : rends l'étape réessayable.

Le type de publication est **manuel** par défaut : après approbation, il faut encore que quelqu'un appuie sur publier.

## Prévois plus d'un tour

Une application a enchaîné quatre refus, une autre trois. En corriger un peut révéler le suivant, et une correction à un endroit peut créer un problème ailleurs. **Après chaque correction, relis toute la liste de `05-refus.md`**, pas seulement le point que tu as changé.
