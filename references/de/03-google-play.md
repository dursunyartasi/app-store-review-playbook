# Android-Build und Google Play

Play ist nachsichtiger als App Review, blockiert Releases aber eher wegen Formularen als wegen Code — und diese Blockaden trifft man leicht.

## Einmalige Einrichtung

**Die App im ersten Durchgang in der Play Console anzulegen, geht nur manuell.** Es gibt keinen API-Weg.

Zwei unumkehrbare Entscheidungen:
- **„Kostenlos" lässt sich nach der Veröffentlichung nicht auf kostenpflichtig umstellen.**
- Der Paketname ist dauerhaft, wie eine iOS-Bundle-ID.

## Signierung und die SHA-1, über die alle stolpern

Du signierst mit einem **Upload-Schlüssel**; Play signiert mit seinem eigenen **App-Signaturschlüssel** neu, den es erst nach deinem ersten AAB-Upload erzeugt.

```bash
keytool -genkey -v -keystore ~/app-release-key.jks \
  -keyalg RSA -keysize 2048 -validity 10000 -alias app
```

Halte den Keystore aus dem Repository heraus. Danach:

**Wenn du Google Sign-In nutzt, brauchst du BEIDE SHA-1-Fingerabdrücke im Android-OAuth-Client** — deinen Upload-Schlüssel *und* Plays App-Signaturschlüssel (Play Console → App-Signatur). Fehlt der zweite, bricht Google Sign-In genau im Play-Build, während dein lokaler Build funktioniert. Auf einem Emulator lässt es sich auch nicht testen, weil die Debug-SHA-1 ebenfalls nicht registriert ist. Der Funktionstest verlangt einen von Play signierten Build auf einem Gerät.

## Bauen

```bash
export ANDROID_HOME=/opt/homebrew/share/android-commandlinetools   # oder dein SDK-Pfad
eas build --platform android --profile production --local --output ./app.aab
```

Ohne `ANDROID_HOME` meldet Gradle „SDK location not found".

**Karten stürzen ohne Schlüssel ab.** `react-native-maps` nutzt auf Android Google Maps und **stürzt bei der nativen Initialisierung ab**, wenn `app.json > android.config.googleMaps.apiKey` fehlt. iOS ist nicht betroffen, weil dort Apple Maps die Voreinstellung ist — genau deshalb rutscht das unbemerkt durch. Prüfe, dass der Schlüssel angekommen ist: AAB entpacken und im Manifest nach `com.google.android.geo.API_KEY` suchen.

## Hochladen

Drag-and-drop funktioniert, bleibt aber für immer manuell; ein typisches AAB ist über 60 MB groß, jenseits jedes Browser-Automatisierungslimits. Automatisiere es mit einem Play-Dienstkonto und `eas.json > submit.android`.

### Release-Fehler, die du sehen wirst

- **„Dieses Release wird bestehenden Nutzern nicht zur Verfügung gestellt, da es ihnen kein Upgrade auf die neu hinzugefügten App-Bundles ermöglicht."** → Version Code erhöhen, oder besser: erst über internen/geschlossenen Test veröffentlichen.
- **„Dieses Release fügt keine App-Bundles hinzu und entfernt keine."** → Das AAB wurde nicht sauber hochgeladen. Version Code prüfen und erneut hochladen.
- **Native Debug-Symbole** müssen eine `native-debug-symbols.zip` mit ABI-Verzeichnissen sein — `armeabi-v7a/`, `arm64-v8a/`, `x86_64/`, jeweils mit `libapp.so` — und **ohne `__MACOSX`- oder `.DS_Store`-Einträge**.

## Erklärungen, die das Release blockieren

**Werbe-ID.** Entpacke das AAB und suche nach `com.google.android.gms.permission.AD_ID`. Firebase Analytics verlangt die Berechtigung und eine passende „wird verwendet"-Erklärung; eine App ohne Werbung sollte beides nicht haben. **Die Regel lautet: Die Erklärung muss exakt zum Manifest passen** — eine Abweichung in beide Richtungen blockiert das Release, und Plays eigener Warntext kann darüber täuschen, welche Seite falsch ist.

**URL der Datenschutzerklärung.** Sie muss 200 zurückgeben. Unsere erste Produktionseinreichung wurde allein deshalb abgelehnt, weil die angegebene URL 404 lieferte; an der App selbst war nichts falsch.

**Formular zur Datensicherheit und Fragebogen zur Inhaltsbewertung.** Beide sind vor der Produktion Pflicht. Beantworte sie nach dem, was die App wirklich tut; sie werden gegen die angegebenen Berechtigungen geprüft.

**Vertriebsländer.** Prüfe sie. Eine unserer Apps lief in der Produktion auf **ein einziges Land** beschränkt, während iOS in 175 Ländern ausgeliefert wurde — kein Zustand, den jemand absichtlich wählt.

## Sensible Berechtigungen

Hintergrundstandort und `FOREGROUND_SERVICE_LOCATION` lösen eine Play-Berechtigungserklärung aus, die ein **Demonstrationsvideo** und eine Prüfung verlangt. Wenn du sie noch nicht brauchst, blockiere sie ausdrücklich, statt sie auszuliefern und festzustecken:

```json
"android": { "blockedPermissions": ["android.permission.ACCESS_BACKGROUND_LOCATION",
                                    "android.permission.FOREGROUND_SERVICE_LOCATION"] }
```

Füge sie später bewusst hinzu, mit vorbereiteter Erklärung und Video.

## Fristen für das Target-API-Level

Play nimmt keine Updates mehr an für Apps, die die Frist zur Anhebung des Target-API-Levels verpassen. Das Datum verschiebt sich jedes Jahr. **Behalte es im Blick** — es am Release-Tag zu erfahren ist ein schlechter Tag.

## Eine Anmerkung zu Plays Tempo

Play genehmigt schnell, und das schneidet in beide Richtungen: Ein kaputtes Release kann binnen etwa einer Stunde live sein und **lässt sich nicht zurückziehen**. Unseres ging mit einem Absturz auf dem Anmeldebildschirm live; das einzige Mittel war, einen korrigierten Version Code nachzuschieben und zu warten. Nutze zuerst den internen Test. Beobachte nach der Veröffentlichung die Absturzzahlen in Play Vitals — so haben wir bestätigt, dass die Korrektur griff (10 Abstürze → 0).
