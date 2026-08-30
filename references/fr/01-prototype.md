# De zéro à un prototype qui tourne

Objectif : une application qui tourne sur le téléphone de la personne, construite de sorte que les exigences des stores décrites dans `05-refus.md` soient déjà satisfaites plutôt que rafistolées après coup.

## Décisions à prendre avant le premier fichier

**Workflow managed ou bare.** Managed garde `ios/` et `android/` générés ; bare les met dans le dépôt. Bare donne le contrôle natif et retire ceci : **`app.json` cesse d'être la source de vérité.** Les chaînes d'usage des permissions ne se propagent plus vers `Info.plist`, et la version vient désormais de `CFBundleShortVersionString` dans `Info.plist` plus `MARKETING_VERSION` dans le pbxproj. Les deux nous ont valu des refus. Commence en managed sauf si un module natif t'y oblige.

**Identifiant de bundle.** Choisis-le maintenant et sûrement : dès qu'un enregistrement App Store Connect existe, **le bundle ID est permanent.** Utilise du DNS inversé sur un domaine que tu contrôles.

**Le nom qu'Apple affichera.** Sur un compte Apple Developer individuel, le nom de développeur affiché sur l'App Store est ton nom légal tel qu'enregistré. Les caractères non ASCII peuvent disparaître silencieusement à l'inscription (les nôtres ont perdu leurs diacritiques turcs), et la correction en libre-service via App Store Connect **ne fonctionne pas** : ce parcours t'entraîne dans la vérification d'adresse et la chaîne du Paid Apps Agreement, et le nom seul ne s'enregistre jamais. La corriger demande une demande au support avec vérification d'identité. **Vérifie l'orthographe caractère par caractère au moment de l'inscription.**

## Échafaudage

```bash
npx create-expo-app@latest my-app
cd my-app
npx expo start            # scanne le QR avec Expo Go
```

Expo Go suffit jusqu'à ce que tu ajoutes un module natif ou aies besoin d'un build signé. Ensuite il faut un development build ou une vraie archive.

## Branche les exigences des stores dès maintenant

Elles sont bon marché le premier jour et chères après un refus.

**Si les utilisateurs peuvent se connecter — suppression de compte (5.1.1(v)).** Elle doit être accessible dans l'application, immédiate et définitive. Pas de désactivation, pas de délai de réflexion, pas de « écrivez au support pour supprimer », pas de redirection vers un site. Redemande le mot de passe, affiche une confirmation destructive, énumère ce qui sera supprimé, et révoque aussi les autorisations tierces du côté du fournisseur.

**Si les utilisateurs peuvent publier — les quatre exigences de la 1.2.** Un élément visible sur chaque message, publication et commentaire (un bouton « ⋯ » ; l'appui long est invisible pour la revue et nous a valu un refus), un blocage qui agit dans les deux sens, un filtrage de contenu sur **tous** les points d'écriture, et une étape de consentement avant qu'un inconnu puisse envoyer plus d'un message privé.

**Les liens juridiques doivent être cliquables (2.1(a)).** « En créant un compte vous acceptez les Conditions » en texte brut, c'est un refus. Fais-en de vrais liens, ouvre-les dans un navigateur intégré plutôt que d'éjecter la personne vers Safari, et place-les aussi sur l'écran de connexion, pas seulement d'inscription.

**Permissions.** Ne demande rien que tu n'utilises pas. Demander l'accès complet à la photothèque avant d'ouvrir le sélecteur nous a valu un refus : le sélecteur iOS moderne renvoie une photo sans aucune permission. Les écrans d'amorce ne doivent pas avoir de libellé orienté : « Continuer », pas « Utiliser ma position ».

**Une adresse de contact qui reçoit vraiment.** Si tu en publies une dans la fiche du store ou les règles intégrées, le domaine a besoin d'un enregistrement MX. Vois le piège du MX dans `06-infrastructure.md` — le nôtre pouvait envoyer mais pas recevoir, si bien que l'adresse de modération de nos règles publiées n'atteignait personne.

## Variables d'environnement

```
.env            → dans .gitignore, comme d'habitude
.easignore      → c'est CELUI-CI que lit EAS, et il remplace .gitignore
```

**Un `.env` ignoré n'atteint jamais l'archive EAS.** Le bundle part avec des variables vides et l'application plante au lancement — **uniquement** en build standalone, si bien que le simulateur et le dev client paraissent parfaits. Dans l'une de nos applications, absolument tous les builds plantaient à cause de ça avant qu'on le trouve. Soit tu configures les variables d'environnement dans EAS, soit tu vérifies que `.easignore` n'exclut pas ce dont le build a besoin.

## Mets-la sur un vrai appareil

Un simulateur ne prouve pas que l'application fonctionne. Les plantages qui n'apparaissent qu'en standalone sont précisément ceux qui arrivent jusqu'à la revue :

```bash
npx expo export --platform ios --output-dir /tmp/exportcheck   # attrape tôt les erreurs d'import
```

Ensuite compile et installe par câble, et suis le log avec `devicectl --console`. Un module natif chargé par `import()` dynamique mais non installé reste invisible en développement — Metro le sert — et plante en standalone avec `RCTFatalException: Cannot find module`.

## Avant d'aller plus loin

Lance `npx tsc --noEmit` et tes tests, et rends-les propres. À partir d'ici, chaque cycle de build coûte de 5 à 40 minutes et, une fois en revue, des jours.

Suite : `02-testflight-ios.md` ou `03-google-play.md`.
