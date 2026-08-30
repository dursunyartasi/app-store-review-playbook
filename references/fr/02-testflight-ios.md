# Build iOS et TestFlight

La boucle qu'on répète à chaque build. Entre 15 et 40 minutes ; la plupart des pièges ci-dessous coûtent un tour complet quand on les rate.

## Vérifie ces deux choses avant de compiler

Les deux sont peu coûteuses à vérifier et chères à découvrir après coup.

1. **Ce numéro de build est-il déjà pris ?** `GET /v1/builds?filter[app]={id}` — un doublon est refusé à l'envoi.
2. **La chaîne de version actuelle est-elle déjà publiée ?** Si la version sur l'App Store est en `READY_FOR_SALE`, son train de publication est fermé et l'envoi échoue avec **90186** / **ITMS-90062**. Il faut monter la **chaîne de version**, pas seulement le numéro de build, et recompiler : la version est compilée dans l'IPA.

Sur une sortie, on y a perdu quatre builds.

## Où vit réellement la version

- **Workflow managed :** `app.json` → `expo.version` et `expo.ios.buildNumber`.
- **Workflow bare :** `ios/<App>/Info.plist` → `CFBundleShortVersionString` et `CFBundleVersion`, plus `MARKETING_VERSION` dans le pbxproj. **`app.json` est ignoré.**

Si tu modifies `app.json` par script, parse puis re-sérialise (`json.load` / `json.dump`). Lire et écrire par le même descripteur tronque le fichier — ça nous a coûté un build.

## Compiler

```bash
npx tsc --noEmit
npx expo export --platform ios --output-dir /tmp/exportcheck   # erreurs d'import avant l'archive

LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8 \
  eas build --platform ios --profile production --local \
  --non-interactive --output ./app-buildNN.ipa
```

Le préfixe `LANG` n'est pas optionnel. Avec Ruby 4.0 et CocoaPods 1.16, `pod install` meurt sur `Unicode Normalization not appropriate for ASCII-8BIT`, surtout s'il y a des caractères non ASCII dans le projet.

`xcodebuild` en direct fonctionne aussi et c'est la sortie de secours quand les identifiants EAS sont périmés :

```bash
LANG=en_US.UTF-8 xcodebuild -workspace App.xcworkspace -scheme App \
  -configuration Release -archivePath /tmp/AppNN.xcarchive \
  -allowProvisioningUpdates \
  -authenticationKeyPath ~/.appstoreconnect/private_keys/AuthKey_<KEY_ID>.p8 \
  -authenticationKeyID <KEY_ID> -authenticationKeyIssuerID <ISSUER_ID> archive
```

Cherche `ARCHIVE SUCCEEDED`, puis `-exportArchive` avec un `ExportOptions.plist` réglé sur `method=app-store` et `signingStyle=automatic`.

### Pendant la compilation

**Ne modifie pas de fichiers source pendant une archive.** Metro embarque un bundle écrit à moitié et l'application plante au lancement. Ce n'est pas théorique : ça nous a coûté un build et ressemblait à un plantage mystérieux.

## Envoyer

```bash
xcrun altool --upload-app -f ./app-buildNN.ipa -t ios \
  --apiKey <KEY_ID> --apiIssuer <ISSUER_ID>
```

Place le `.p8` dans `~/.appstoreconnect/private_keys/AuthKey_<KEY_ID>.p8` ; altool l'y trouve. Environ 15 secondes.

**Préfère ça à `eas submit`.** On a vu `eas submit` rester bloqué 23 minutes sans sortie ni envoi, et échouer sur « Unable to upload archive. Failed to authenticate for session ». altool donne la vraie erreur.

## Après l'envoi

**`UPLOAD SUCCEEDED` ne signifie pas accepté.** Interroge l'API ASC jusqu'à ce que le build soit `VALID` ; Apple refuse aussi pendant le traitement. Une fois valide, fais expirer le build précédent pour que les testeurs ne voient que le nouveau :

```
PATCH /v1/builds/{id}   {"expired": true}
```

## Erreurs d'envoi à savoir reconnaître

| Code | Signification |
|---|---|
| **90062** / ITMS-90062 | Cette version est déjà publiée — monte la chaîne de version |
| **90186** | Train de pré-publication fermé — même cause |
| **90713** | Un target n'a pas de `CFBundleIconName` — Watch et widgets ont besoin de leur propre icône |
| **ITMS-90863** | Avertissement de symboles Apple Silicon. **Normal pour les apps Expo, ce n'est pas un refus.** Ignore-le. |

## Targets supplémentaires

Les targets Watch et Live Activity ont besoin de leurs propres profils de provisionnement dans `credentials.json`, tous sous le même certificat de distribution. Archiver un target Watch exige la plateforme watchOS **pour appareil** sur le Mac — le SDK du simulateur ne suffit pas :

```bash
xcodebuild -downloadPlatform watchOS    # ~4 Go
```

**Teste les parcours propres au target supplémentaire.** La connexion sur notre target Watch n'avait jamais fonctionné — elle envoyait `email` là où le backend lisait `identifier` — et Apple l'a trouvée avant nous, dans un refus 2.1.

## Quand Xcode se met à jour en cours de projet

Une mise à jour automatique laisse les builds échouer sur `iOS <version> Platform Not Installed`, même si tout compilait le matin même :

```bash
xcodebuild -downloadPlatform iOS   # ~8,5 Go, sudo inutile
xcodebuild -runFirstLaunch
```

## Entretien

Les temporaires d'EAS grossissent sans limite — chez nous `/var/folders/.../eas-cli-nodejs` a atteint 35 Go. Un disque plein fait échouer le build sur `No space left`. Nettoie entre les sorties. Que les numéros de build sautent après des tentatives ratées est normal.

Suite : `04-soumission-app-store.md`.
