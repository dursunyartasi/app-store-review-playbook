# Catalogue des refus et check-list avant soumission

Ce ne sont pas des conseils génériques. Chaque point ci-dessous est réellement arrivé en publiant huit applications, avec la cause profonde que nous avons fini par trouver — et qui n'était souvent pas ce que disait l'avis de refus. Les applications sont anonymisées : **Application A** (événements sociaux), **Application B** (guide de lieux et cartes), **Application C** (outil B2B pour agences).

---

## La leçon la plus coûteuse : ce qu'il ne faut PAS écrire dans les notes d'App Review

L'application A, build 49, a été refusée au titre de la **Guideline 4.2 (Minimum Functionality)**. Le texte du refus citait, presque mot pour mot, les phrases que nous avions nous-mêmes écrites dans les notes de revue :

> "We are targeting a deliberately **small**, curated early community — **a few dozen** invited members, **not a mass-market** social network. **Staying small is a core part of the product.**"

Apple a lu cela comme « ceci relève de la distribution Ad Hoc, pas de l'App Store ».

**Règle : ne présente jamais ton application ainsi** — « small », « niche », « few dozen », « invite-only », « not mass-market », « private group », « closed community », « pour une communauté précise ».

**Cadrage correct :** « ouverte à tous, gratuite, téléchargeable depuis n'importe quelle ville ou pays ; la couche d'adhésion curée est *optionnelle* ». Même si le produit est réellement sur invitation, il lui faut un visage qui fonctionne **sans compte**, et c'est ce visage que les notes doivent décrire.

Par ailleurs : **les notes de revue sont limitées à 4000 caractères.** Au-delà, l'API renvoie 409.

---

## Check-list avant soumission

Chaque case vient d'un refus. Vérifie-les une par une.

### Comptes et données (5.1.1)
- [ ] **Suppression de compte.** Si l'on peut créer un compte, on doit pouvoir le supprimer depuis l'application — 5.1.1(v). *(A causé des refus dans deux de nos applications.)* Le parcours qu'Apple a fini par accepter remplissait tout ceci, et chaque point compte :
  - **Immédiate et définitive.** Pas de désactivation, pas de délai de grâce.
  - **Pas d'étape par le support, l'e-mail ou le téléphone, et pas de redirection vers le web** — Apple considère cela comme « non supprimable depuis l'application ».
  - Ré-authentification par mot de passe et confirmation destructive.
  - Une liste explicite de ce qui sera supprimé ; les autorisations tierces (Instagram/Facebook) révoquées aussi côté fournisseur.
- [ ] **Chaînes d'usage des permissions** — 5.1.1(ii). **Piège du workflow bare :** si le répertoire `ios/` est dans le dépôt, les chaînes de `app.json` **ne** se propagent **pas** vers `Info.plist`. Vérifie à la main.
- [ ] **Ne demande pas de permissions que tu n'utilises pas.** L'application B a été refusée au titre de la 5.1.1 parce que `pickImage` demandait l'accès complet à la photothèque avant d'ouvrir le sélecteur. Le PHPicker moderne d'iOS renvoie une photo **sans aucune permission** → supprime l'appel.
- [ ] **Liens juridiques cliquables** — 2.1(a). L'application C a échoué ici : la phrase « en créant un compte vous acceptez les Conditions » était du **texte brut**, et l'écran de connexion n'avait aucun lien, si bien que la revue n'a pas pu voir les conditions. Ouvre-les dans un **navigateur intégré** (`expo-web-browser`) plutôt que d'éjecter vers Safari, et place-les aussi sur l'écran de connexion.
- [ ] **Écran d'amorce de localisation** — 5.1.1(iv). Pas de libellé orienté : « Utiliser ma position » ❌ → « Continuer » ✅. Et pas deux « Passer » qui sèment la confusion.

### Contenu généré par les utilisateurs (1.2) — obligatoire dès que l'on peut publier
L'application A, build 51, a échoué ici. Les quatre sont nécessaires :
- [ ] Un élément de signalement **visible**. Un appui long **ne suffit pas** : la revue ne le trouvera pas. Mets un bouton « ⋯ » visible à côté de chaque message, publication et commentaire.
- [ ] Le blocage (dans les deux sens : la personne bloquée ne peut pas non plus t'écrire).
- [ ] Le filtrage de contenu sur **tous** les points d'écriture, pas seulement les évidents.
- [ ] Le consentement en messagerie privée : celui qui initie ne peut envoyer qu'un message tant que l'autre n'a pas accepté.

### Signaux d'achat (3.1.1) — le piège le plus retors des applications gratuites et B2B
L'application C a été refusée sur cette guideline **deux fois**.
- [ ] **Ne laisse aucun prix, nom de formule, compteur de crédits, mur payant, bouton d'upgrade ni lien d'achat externe.** Ce qui a coulé le build 25, c'est une ligne « Intelligence n'est pas dans votre formule actuelle — Solo ou supérieur requis », un compteur de crédits restant et une étiquette « 1 crédit par marque ». Afficher un simple nom de formule suffit.
- [ ] **Ne t'appuie pas uniquement sur l'argument 3.1.3(f) « Free Stand-alone Apps » — Apple l'a rejeté.** On l'a tenté au build 26.
- [ ] **Un écran d'inscription public détruit un argument B2B.** C'était le maillon faible : un écran « rejoindre » se lit comme de la vente en libre-service au consommateur et contredit directement le « only sold directly by you to organizations » de la 3.1.3(c). La solution au build 27 a été de **supprimer entièrement l'écran d'inscription** — l'application ne propose que la connexion.

### Métadonnées
- [ ] **Classification par âge** — 2.3.6. Toute application avec une dimension « rencontrer des gens / réseautage » a besoin de `matureOrSuggestiveThemes` au moins en `INFREQUENT_OR_MILD`. Corrigible par API : `PATCH /v1/ageRatingDeclarations/{id}`.
- [ ] **L'URL de la politique de confidentialité renvoie-t-elle 200 ?** Notre première soumission en production sur Play a été refusée uniquement parce que l'URL déclarée renvoyait 404.

### Ce que la revue verra réellement
- [ ] **Le compte de démonstration fonctionne-t-il vraiment ?** Teste sur un appareil. L'application A a été refusée au titre de la 2.1 parce que la connexion depuis le target Apple Watch n'avait jamais fonctionné : elle envoyait `email` alors que le backend lisait `identifier`, d'où un 422. Personne ne l'avait vu parce que le target téléphone envoyait le bon champ.
- [ ] **Les données de démo sont-elles fraîches ?** Dans l'application A, 16 événements initialisés étaient dans le passé : la revue aurait ouvert une application vide. Garde un script idempotent qui décale les dates.
- [ ] **Un mur de vérification piège-t-il la revue ?** Si une personne inscrite mais non vérifiée ne voit rien, l'application paraît fermée. Laisse les invités parcourir ; n'exige la vérification qu'à l'écriture.
- [ ] **Ne publie pas de modules vides ou désactivés.** Un feature flag éteint dans l'application A laissait une section « Cours » vide qui a provoqué deux refus 2.1 d'affilée (App Completeness, puis Information Needed). Elle a fini supprimée. **Ne soumets pas une fonctionnalité à moitié faite — retire-la.**
- [ ] **Tu ne choisis pas l'appareil de revue.** Chez nous, les retours sont venus d'un iPad Air et d'une Apple Watch. Teste aussi les targets autres que le principal.

### Intégration à la plateforme (4.0 Design)
- [ ] **Ta fonction de cartes ou de localisation est-elle intégrée à l'application native ?** L'application B a été refusée au titre de la 4.0.0 parce qu'elle se contentait de renvoyer vers Google Maps. Propose Apple Maps (`maps.apple.com`) comme option.

### Android
- [ ] **La déclaration d'identifiant publicitaire correspond-elle au manifeste ?** Décompresse le `.aab` et cherche `com.google.android.gms.permission.AD_ID`. S'il n'y est pas, la déclaration doit dire « non utilisé » : une déclaration erronée bloque la publication.
- [ ] **Pays de distribution.** Une de nos applications s'est retrouvée par accident limitée à **un seul pays** en production alors qu'iOS était distribué dans 175.
- [ ] **La localisation en arrière-plan** peut déclencher la déclaration de permission de localisation de Play.
- [ ] **Clé d'API Maps** dans `app.json > android.config.googleMaps.apiKey` : sans elle, `react-native-maps` **plante à l'initialisation native sur Android**. Sur iOS rien ne se passe (Apple Maps y est par défaut), et c'est précisément pour ça que ça passe inaperçu.
- [ ] **La connexion Google exige DEUX SHA-1 :** ta clé d'importation **et** la clé de signature d'application de Play. Play ne génère la sienne qu'après le premier envoi d'AAB ; si ce SHA-1 n'est pas ajouté au client OAuth Android, la connexion Google est cassée dans le build Play. Impossible à tester sur émulateur non plus (le SHA-1 de debug n'y est pas enregistré) : il faut un build signé par Play sur un appareil.

---

## Catalogue des refus

| Guideline | Ce que le store a dit | Vraie cause profonde | Correctif |
|---|---|---|---|
| **4.2** Minimum Functionality | « small, or niche, set of users » | **Notre propre phrase dans les notes de revue** | Parcours de découverte sans compte + points d'API publics + rééquilibrage du contenu |
| **1.2** UGC | pas de filtrage / signalement / blocage | Le signalement n'existait que derrière un appui long invisible ; absent des messages privés et des salons | Menu « ⋯ » visible sur 8 surfaces, filtre sur 9 points d'écriture, consentement en messagerie |
| **2.1** Compte de démo | connexion impossible | Le target Watch envoyait `email`, le backend lisait `identifier` → 422 | Champ corrigé ; section « WATCH — PLEASE READ FIRST » dans les notes |
| **2.1** App Completeness | « could not access the courses » | Feature flag éteint ; la section s'affichait vide | Fonctionnalité entièrement retirée |
| **2.1** Information Needed | « combien d'utilisateurs visez-vous ? » | Le même module vide plus le cadrage de la 4.2 | Notes réécrites |
| **2.3.6** Classification par âge | « Mature or Suggestive Themes » | Thématique de rencontre non déclarée | `PATCH ageRatingDeclarations` via l'API ASC |
| **3.1.1** Achat intégré | signaux d'achat dans une application gratuite | Nom de formule, compteur de crédits, texte « formule Solo requise » | Toute trace de prix et de formule retirée |
| **3.1.1** (deuxième fois) | la même guideline à nouveau | L'argument 3.1.3(f) n'a pas suffi ; un **écran d'inscription public** contredisait notre argument B2B | Écran d'inscription supprimé, connexion seule |
| **2.1(a)** App Completeness | conditions non consultables | Le texte juridique était brut, non cliquable, et absent de l'écran de connexion | Liens cliquables ouvrant dans un navigateur intégré |
| **5.1.1(v)** Data Collection | pas de suppression de compte | — | Suppression de compte dans l'application |
| **5.1.1(ii)** | chaîne d'usage manquante | Le workflow bare ne synchronise pas `app.json` → `Info.plist` | Modifier `Info.plist` directement |
| **5.1.1(iv)** Parcours de localisation | écran d'amorce orienté | Libellés de boutons et double option de saut | « Continuer », une seule sortie |
| **5.1.1** Accès aux photos | demandait la permission photothèque | Le PHPicker n'a jamais eu besoin de cette permission | Appel de permission supprimé |
| **4.0.0** Design | « not integrated with built-in mapping » | Renvoi exclusif vers Google Maps | Option Apple Maps ajoutée |
| **Play** (production) | soumission refusée | L'URL déclarée de politique de confidentialité renvoyait 404 | Alias permanent et entrée corrigée dans la console |

**Remarque :** corriger un refus peut inviter le suivant. Une application a enchaîné quatre refus, une autre trois. Après chaque correction, reprends la liste **entière**.

---

## Pièges de l'API App Store Connect et des déclarations

Trouvés en remplissant les champs de soumission par API :

- **Le type de capture `APP_IPHONE_69` n'existe pas.** Le plus grand type iPhone accepté par l'API est `APP_IPHONE_67` (1290×2796). Les images en 1320×2868 pour l'appareil 6,9" sont **refusées** : envoie du 6,7" et laisse Apple agrandir.
- **`whatsNew` ne peut pas être modifié sur une première version** — 409, « cannot be edited at this time ». Il n'existe que pour les mises à jour.
- **Les types de champs de la classification par âge sont mélangés :** certains BOOLEAN (`messagingAndChat`, `userGeneratedContent`, `advertising`), d'autres énumérations STRING (`contests`, `profanityOrCrudeHumor` → `NONE` / `INFREQUENT_OR_MILD` / `FREQUENT_OR_INTENSE`). Le mauvais type renvoie 409, et le message nomme le bon ensemble.
- **Apple a changé les tranches d'âge en 2025 : le 12+ n'existe plus.** Ce sont 4+, 9+, 13+, 16+, 18+. Des réponses honnêtes peuvent donner 4+ ; remonte avec `ageRatingOverrideV2` (par exemple `THIRTEEN_PLUS`).

**Déclaration App Privacy :**
- **Un numéro de pièce d'identité n'est pas une « Sensitive Info ».** La catégorie sensible d'Apple couvre l'origine, la religion, l'orientation sexuelle, la biométrie et assimilés ; une pièce d'identité n'y figure pas → la bonne case est **« Other Data Types »**.
- **Les données bancaires dans ta propre base sont « Collected ».** Apple n'exonère que si le prestataire de paiement les détient et que tu n'y as pas accès.
- ⚠️ **Piège du clic à l'aveugle :** l'assistant s'affiche à des hauteurs différentes selon le type de donnée. Répéter le même point de clic a produit des réponses comme « L'identifiant utilisateur sert au suivi : OUI », qui étaient fausses. Vérifie par capture l'état final de chaque élément.

---

## Pièges de build et d'envoi

### Le train de publication
**Tu ne peux pas envoyer un nouveau build sur une chaîne de version déjà approuvée** — erreurs altool 90062 / 90186 (« Invalid Pre-Release Train ... closed »). Monte `version` dans `app.json` et **recompile** : la chaîne de version est dans l'IPA. On y a brûlé un build entier.

### Envoi
- `eas submit` peut se bloquer (plus de 23 minutes, sans sortie) ou échouer sur « Failed to authenticate for session ». **La voie fiable, c'est altool en direct :**
  ```bash
  xcrun altool --upload-app -f build.ipa -t ios --apiKey <KEY_ID> --apiIssuer <ISSUER_ID>
  ```
  Place le `.p8` dans `~/.appstoreconnect/private_keys/AuthKey_<KEY_ID>.p8`. Environ 15 secondes.
- **« Upload succeeded » n'est pas « accepté ».** Apple peut encore refuser pendant le traitement. Interroge jusqu'à `VALID`, puis fais expirer le build précédent (`PATCH /v1/builds/{id}` avec `{"expired": true}`).
- **Les targets Watch et widget ont besoin d'une icône** (`CFBundleIconName`), sinon Apple refuse l'envoi avec l'erreur **90713**.
- **ITMS-90062** signifie « cette version est déjà publiée » : monte la chaîne de version.
- **ITMS-90863** (avertissement de symboles Apple Silicon) est **normal pour les applications Expo et ne provoque pas de refus.** Ne cours pas après.

### Ordre de la resoumission
1. **Deux versions ne peuvent pas être en revue en même temps.** Annule la `reviewSubmission` existante (`canceled=true`) et attends CANCELING → COMPLETE.
2. La version passe en `DEVELOPER_REJECTED` (modifiable) → PATCH de la chaîne de version → PATCH de la relation avec le build.
3. ⚠️ **Piège de l'échange :** juste après l'annulation, l'attachement renvoie 409, et si le script continue quand même, il soumet l'**ancien** build. Réessaie l'attachement et **vérifie le build attaché avant de soumettre** (`GET /appStoreVersions/{id}/build`).
4. ⚠️ `POST reviewSubmissionItems` peut renvoyer 409 `ENTITY_STATE_INVALID` pendant la transition. Ça passe quelques secondes plus tard : rends-le réessayable.

### Environnement de build local
- **Si Xcode se met à jour en cours de session**, les builds échouent sur « iOS X Platform Not Installed ». Solution : `xcodebuild -downloadPlatform iOS` (~8,5 Go, sans sudo) et `xcodebuild -runFirstLaunch`. Que ça ait compilé le matin même ne prouve rien sur l'état actuel.
- **CocoaPods sous Ruby 4.0 :** `pod install` meurt sur `Unicode Normalization not appropriate for ASCII-8BIT`. Lance-le avec `LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8`.
- **Modular headers dans le Podfile :** GoogleSignIn 9.2.0 a besoin de `:modular_headers => true` pour `AppCheckCore`, `GoogleUtilities` et `RecaptchaInterop`.
- **Un profil de provisionnement antérieur à une nouvelle capability** fait échouer le build local. EAS non interactif ne met pas à jour les identifiants : soit tu les renouvelles en interactif, soit tu passes par l'API ASC.
- **Les capabilities Apple Developer s'activent par API** (sans passer par le portail) : `POST /v1/bundleIdCapabilities`. Sans l'objet `settings`, ça renvoie 409.
- **`ANDROID_HOME` est obligatoire** pour les builds Android locaux, sinon Gradle signale « SDK location not found ».
- **Ne modifie jamais de fichiers source pendant qu'une archive compile** — Metro embarque un bundle à moitié écrit et l'application plante au lancement.
- **Les temporaires d'EAS grossissent sans limite** (les nôtres ont atteint 35 Go). Nettoie ; un disque plein fait échouer le build sur « No space left ».
- Que les numéros de build sautent après des tentatives ratées est normal.

### Erreurs de publication Play
- **« Cette version ne sera pas disponible pour les utilisateurs existants… »** → monte le version code, ou publie d'abord par un test interne ou fermé.
- **« Cette version n'ajoute ni ne supprime aucun app bundle. »** → l'AAB n'a pas été envoyé proprement ; vérifie le version code et renvoie-le.
- **Les symboles de debug natifs** doivent aller dans un `native-debug-symbols.zip` avec des répertoires par ABI (`armeabi-v7a/`, `arm64-v8a/`, `x86_64/`, chacun avec `libapp.so`) et sans entrées `__MACOSX` ni `.DS_Store`.
- ⚠️ **Échéances du niveau d'API cible.** Play bloque la publication des mises à jour pour les applications qui ratent l'échéance. Garde la date à l'œil.
- **La nuance AD_ID :** Firebase Analytics exige la permission dans le manifeste et une déclaration « utilisé » ; une application sans publicité ne doit avoir ni l'une ni l'autre. **La règle est que la déclaration corresponde exactement au manifeste** — un écart dans un sens ou dans l'autre bloque la publication.

### Plantages qui n'existent qu'en build standalone
- **Les simulateurs et dev clients ne les attrapent pas.** Teste sur un vrai appareil par câble avec `devicectl --console`.
- Si `.env` est dans `.gitignore`, il n'atteint jamais l'archive EAS : variables vides dans le bundle et plantage au lancement. Dans une application, *tous* les builds plantaient à cause de ça.
- Un module natif importé dynamiquement mais non installé reste invisible en développement (Metro le sert) et plante en standalone sur `RCTFatalException: Cannot find module`.
- **Hermes stocke les chaînes en UTF-16.** Chercher des chaînes non ASCII dans le bundle en UTF-8 ne renvoie rien : vérifie en UTF-16.

---

## Enregistrement en boutique — une seule fois, et manuellement

- **L'enregistrement de l'application dans App Store Connect ne peut pas être créé par API.** On a essayé et confirmé. Fais-le dans le navigateur.
- **Créer l'application dans la Play Console est également manuel** la première fois.
- **Le bundle ID se lie définitivement à l'enregistrement et ne peut pas changer.**
- **Choisir « Gratuite » sur Play est irréversible** : impossible de passer au payant après publication.
- ⚠️ **Les caractères non ASCII peuvent disparaître à l'inscription.** Sur un compte Apple individuel, le nom de développeur affiché sur l'App Store est ton nom légal ; le nôtre a perdu ses diacritiques à l'inscription. Le corriger via App Store Connect → Business → Legal Entity **ne fonctionne pas** : ce parcours t'entraîne dans la vérification d'adresse et la chaîne du Paid Apps Agreement, et le nom seul ne s'enregistre pas. La voie qui marche est le support Apple → « Membership & Account » → correction du nom légal, avec vérification d'identité. **Vérifie l'orthographe caractère par caractère à l'inscription.**

## Limites pour un assistant IA qui exécute ce guide

- **Ne saisis jamais le mot de passe Apple ou Google ni le code 2FA.** App Store Connect exige sa propre connexion (la session du portail développeur ne s'y reporte pas). Le déroulé : la personne se connecte elle-même, confirme, et l'assistant reprend à partir de là les étapes API et console.
- **L'envoi de fichiers par navigateur est plafonné à 10 Mo ;** un `.aab` typique dépasse 60 Mo. Laisse la personne l'envoyer, ou automatise avec un compte de service Play et `eas.json > submit.android`.
- **Ne coche jamais de cases de déclaration ou de consentement** sans l'accord explicite de la personne.

---

Quand un nouveau refus arrive, trouve d'abord la cause profonde, puis ajoute une ligne ici. Un guide ne vaut que ce que le dernier incident lui a appris.
