# Build Android et Google Play

Play est plus indulgent qu'App Review, mais il bloque les publications sur la paperasse plutôt que sur le code — et ces blocages sont faciles à rencontrer.

## Configuration initiale

**Créer l'application dans la Play Console se fait manuellement la première fois.** Il n'y a pas de voie par API.

Deux choix irréversibles :
- **« Gratuite » ne peut pas devenir payante après publication.**
- Le nom de package est permanent, comme un bundle ID iOS.

## Signature, et le SHA-1 qui piège tout le monde

Tu signes avec une **clé d'importation** ; Play resigne avec sa propre **clé de signature d'application**, qu'il ne génère qu'après ton premier envoi d'AAB.

```bash
keytool -genkey -v -keystore ~/app-release-key.jks \
  -keyalg RSA -keysize 2048 -validity 10000 -alias app
```

Garde le keystore hors du dépôt. Ensuite :

**Si tu utilises la connexion Google, il te faut LES DEUX empreintes SHA-1 sur le client OAuth Android** — ta clé d'importation *et* la clé de signature d'application de Play (Play Console → Signature d'application). Rate la seconde et la connexion Google casse précisément dans le build Play, alors que ton build local fonctionne. Impossible à tester sur émulateur non plus, car le SHA-1 de debug n'est pas enregistré. Le test fonctionnel exige un build signé par Play sur un appareil.

## Compiler

```bash
export ANDROID_HOME=/opt/homebrew/share/android-commandlinetools   # ou ton chemin de SDK
eas build --platform android --profile production --local --output ./app.aab
```

Sans `ANDROID_HOME`, Gradle signale « SDK location not found ».

**Les cartes plantent sans clé.** `react-native-maps` utilise Google Maps sur Android et **plante à l'initialisation native** si `app.json > android.config.googleMaps.apiKey` manque. iOS n'est pas touché car Apple Maps y est la valeur par défaut — c'est exactement pour ça que ça passe en production sans qu'on le remarque. Vérifie que la clé est bien passée : décompresse l'AAB et cherche `com.google.android.geo.API_KEY` dans le manifeste.

## Envoyer

Le glisser-déposer marche mais reste manuel pour toujours ; un AAB typique dépasse 60 Mo, au-delà de toute limite d'automatisation navigateur. Automatise avec un compte de service Play et `eas.json > submit.android`.

### Erreurs de publication que tu verras

- **« Cette version ne sera pas disponible pour les utilisateurs existants car elle ne leur permet pas de passer aux nouveaux app bundles ajoutés. »** → monte le version code, ou mieux, publie d'abord par un test interne ou fermé.
- **« Cette version n'ajoute ni ne supprime aucun app bundle. »** → l'AAB n'a pas été envoyé proprement. Vérifie le version code et renvoie-le.
- **Les symboles de debug natifs** doivent être un `native-debug-symbols.zip` avec des répertoires par ABI — `armeabi-v7a/`, `arm64-v8a/`, `x86_64/`, chacun contenant `libapp.so` — et **sans entrées `__MACOSX` ni `.DS_Store`**.

## Déclarations qui bloquent la publication

**Identifiant publicitaire.** Décompresse l'AAB et cherche `com.google.android.gms.permission.AD_ID`. Firebase Analytics exige la permission et une déclaration « utilisé » correspondante ; une application sans publicité ne devrait avoir ni l'une ni l'autre. **La règle est que la déclaration corresponde exactement au manifeste** — un écart dans un sens ou dans l'autre bloque la publication, et l'avertissement de Play lui-même peut induire en erreur sur le côté fautif.

**URL de la politique de confidentialité.** Elle doit renvoyer 200. Notre première soumission en production a été refusée uniquement parce que l'URL déclarée renvoyait 404 ; rien d'autre ne clochait dans l'application.

**Formulaire de sécurité des données et questionnaire de classification.** Les deux sont obligatoires avant la production. Réponds selon ce que l'application fait réellement ; ils sont recoupés avec les permissions déclarées.

**Pays de distribution.** Vérifie-les. Une de nos applications tournait en production limitée à **un seul pays** alors qu'iOS était distribué dans 175 — un état que personne ne choisit délibérément.

## Permissions sensibles

La localisation en arrière-plan et `FOREGROUND_SERVICE_LOCATION` déclenchent une déclaration de permission Play qui exige une **vidéo de démonstration** et une revue. Si tu n'en as pas encore besoin, bloque-les explicitement plutôt que de les publier et rester coincé :

```json
"android": { "blockedPermissions": ["android.permission.ACCESS_BACKGROUND_LOCATION",
                                    "android.permission.FOREGROUND_SERVICE_LOCATION"] }
```

Ajoute-les plus tard délibérément, avec la déclaration et la vidéo prêtes.

## Échéances du niveau d'API cible

Play cesse d'accepter les mises à jour des applications qui ratent l'échéance de relèvement du niveau d'API cible. La date bouge chaque année. **Garde-la à l'œil** — l'apprendre le jour de la sortie est une mauvaise journée.

## Une note sur la vitesse de Play

Play approuve vite, et ça coupe des deux côtés : une version cassée peut être en ligne en une heure environ et **ne peut pas être retirée**. La nôtre est sortie avec un plantage sur l'écran de connexion ; le seul remède a été de pousser un version code corrigé et d'attendre. Utilise d'abord le test interne. Après publication, surveille le nombre de plantages dans Play Vitals — c'est comme ça qu'on a confirmé que le correctif avait pris (10 plantages → 0).
