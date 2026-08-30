# iOS-Build und TestFlight

Der Zyklus, den wir bei jedem Build durchlaufen. Etwa 15 bis 40 Minuten; die meisten Fallen unten kosten einen ganzen Durchlauf, wenn man sie übersieht.

## Prüfe diese zwei Dinge vor dem Bauen

Beide sind billig zu prüfen und teuer, wenn man sie hinterher entdeckt.

1. **Ist diese Build-Nummer schon belegt?** `GET /v1/builds?filter[app]={id}` — ein Duplikat wird beim Upload abgelehnt.
2. **Ist die aktuelle Versionszeichenkette bereits veröffentlicht?** Steht die Version im App Store auf `READY_FOR_SALE`, ist ihr Release-Zug geschlossen und der Upload scheitert mit **90186** / **ITMS-90062**. Du musst die **Versionszeichenkette** anheben, nicht nur die Build-Nummer, und neu bauen: Die Version ist ins IPA kompiliert.

Bei einem Release haben wir daran vier Builds verloren.

## Wo die Version tatsächlich wohnt

- **Managed Workflow:** `app.json` → `expo.version` und `expo.ios.buildNumber`.
- **Bare Workflow:** `ios/<App>/Info.plist` → `CFBundleShortVersionString` und `CFBundleVersion`, dazu `MARKETING_VERSION` im pbxproj. **`app.json` wird ignoriert.**

Wenn du `app.json` per Skript bearbeitest, parse und serialisiere neu (`json.load` / `json.dump`). Lesen und Schreiben über denselben Handle kürzt die Datei — das hat uns einen Build gekostet.

## Bauen

```bash
npx tsc --noEmit
npx expo export --platform ios --output-dir /tmp/exportcheck   # Import-Fehler vor dem Archive

LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8 \
  eas build --platform ios --profile production --local \
  --non-interactive --output ./app-buildNN.ipa
```

Das `LANG`-Präfix ist nicht optional. Unter Ruby 4.0 mit CocoaPods 1.16 stirbt `pod install` mit `Unicode Normalization not appropriate for ASCII-8BIT`, besonders wenn irgendwo im Projekt Nicht-ASCII-Zeichen stehen.

Direktes `xcodebuild` funktioniert ebenfalls und ist der Notausgang, wenn die EAS-Anmeldedaten veraltet sind:

```bash
LANG=en_US.UTF-8 xcodebuild -workspace App.xcworkspace -scheme App \
  -configuration Release -archivePath /tmp/AppNN.xcarchive \
  -allowProvisioningUpdates \
  -authenticationKeyPath ~/.appstoreconnect/private_keys/AuthKey_<KEY_ID>.p8 \
  -authenticationKeyID <KEY_ID> -authenticationKeyIssuerID <ISSUER_ID> archive
```

Achte auf `ARCHIVE SUCCEEDED`, dann `-exportArchive` mit einer `ExportOptions.plist` mit `method=app-store` und `signingStyle=automatic`.

### Während gebaut wird

**Bearbeite während eines Archive keine Quelldateien.** Metro bettet ein halb geschriebenes Bundle ein und die App stürzt beim Start ab. Das ist nicht theoretisch: Es kostete uns einen Build und sah aus wie ein rätselhafter Absturz.

## Hochladen

```bash
xcrun altool --upload-app -f ./app-buildNN.ipa -t ios \
  --apiKey <KEY_ID> --apiIssuer <ISSUER_ID>
```

Leg die `.p8` nach `~/.appstoreconnect/private_keys/AuthKey_<KEY_ID>.p8`; altool findet sie dort. Dauert etwa 15 Sekunden.

**Bevorzuge das gegenüber `eas submit`.** Wir haben `eas submit` 23 Minuten ohne Ausgabe und ohne Upload hängen sehen und mit „Unable to upload archive. Failed to authenticate for session" scheitern sehen. altool nennt den echten Fehler.

## Nach dem Upload

**`UPLOAD SUCCEEDED` heißt nicht angenommen.** Frage die ASC-API ab, bis der Build `VALID` ist; Apple lehnt auch während der Verarbeitung ab. Sobald er gültig ist, setze den vorherigen Build auf abgelaufen, damit Testende nur den neuen sehen:

```
PATCH /v1/builds/{id}   {"expired": true}
```

## Upload-Fehler, die man kennen sollte

| Code | Bedeutung |
|---|---|
| **90062** / ITMS-90062 | Diese Version ist bereits veröffentlicht — Versionszeichenkette anheben |
| **90186** | Pre-Release-Zug geschlossen — dieselbe Ursache |
| **90713** | Einem Target fehlt `CFBundleIconName` — Watch und Widgets brauchen ein eigenes Icon |
| **ITMS-90863** | Apple-Silicon-Symbolwarnung. **Bei Expo-Apps normal, keine Ablehnung.** Ignorieren. |

## Zusätzliche Targets

Watch- und Live-Activity-Targets brauchen eigene Provisioning Profiles in `credentials.json`, alle unter demselben Distributionszertifikat. Das Archivieren eines Watch-Targets erfordert die watchOS-**Geräteplattform** auf dem Mac — das Simulator-SDK reicht nicht:

```bash
xcodebuild -downloadPlatform watchOS    # ~4 GB
```

**Teste die eigenen Abläufe des zusätzlichen Targets.** Die Anmeldung auf unserem Watch-Target hatte nie funktioniert — sie schickte `email`, während das Backend `identifier` las — und Apple fand das vor uns, in einer 2.1-Ablehnung.

## Wenn Xcode mitten im Projekt aktualisiert

Ein Auto-Update lässt Builds mit `iOS <Version> Platform Not Installed` scheitern, obwohl am selben Morgen noch alles lief:

```bash
xcodebuild -downloadPlatform iOS   # ~8,5 GB, kein sudo nötig
xcodebuild -runFirstLaunch
```

## Aufräumen

EAS-Temporärdateien wachsen unbegrenzt — bei uns erreichte `/var/folders/.../eas-cli-nodejs` 35 GB. Eine volle Platte lässt den Build mit `No space left` scheitern. Zwischen Releases aufräumen. Dass Build-Nummern nach fehlgeschlagenen Versuchen springen, ist normal.

Weiter: `04-app-store-einreichung.md`.
