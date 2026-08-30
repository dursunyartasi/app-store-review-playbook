# Ablehnungskatalog und Checkliste vor der Einreichung

Das sind keine allgemeinen Store-Tipps. Jeder Punkt ist beim Veröffentlichen von acht Apps wirklich passiert, mitsamt der Ursache, die wir am Ende fanden — und die war oft nicht das, was in der Ablehnung stand. Die Apps sind anonymisiert: **App A** (soziale Events), **App B** (Orts- und Kartenführer), **App C** (B2B-Werkzeug für Agenturen).

---

## Die teuerste Lektion: was NICHT in die Review Notes gehört

App A, Build 49, wurde nach **Guideline 4.2 (Minimum Functionality)** abgelehnt. Der Ablehnungstext zitierte fast wörtlich die Sätze, die wir selbst in die Review Notes geschrieben hatten:

> "We are targeting a deliberately **small**, curated early community — **a few dozen** invited members, **not a mass-market** social network. **Staying small is a core part of the product.**"

Apple las das als „das gehört in die Ad-Hoc-Verteilung, nicht in den App Store".

**Regel: beschreibe deine App nie so** — „small", „niche", „few dozen", „invite-only", „not mass-market", „private group", „closed community", „für eine bestimmte Community".

**Richtige Rahmung:** „offen für alle, kostenlos, aus jeder Stadt und jedem Land herunterladbar; die kuratierte Mitgliedschaftsebene ist *optional*". Selbst wenn das Produkt wirklich auf Einladung läuft, braucht es ein Gesicht, das **ohne Konto** funktioniert, und genau dieses Gesicht sollten die Notes beschreiben.

Außerdem: **Review Notes sind auf 4000 Zeichen begrenzt.** Darüber liefert die API 409.

---

## Checkliste vor der Einreichung

Jedes Kästchen stammt aus einer Ablehnung. Prüfe eins nach dem anderen.

### Konten und Daten (5.1.1)
- [ ] **Kontolöschung.** Wenn Konten angelegt werden können, müssen sie sich in der App löschen lassen — 5.1.1(v). *(Verursachte Ablehnungen in zwei unserer Apps.)* Der Ablauf, den Apple schließlich akzeptierte, erfüllte all das, und jeder Punkt zählt:
  - **Sofort und dauerhaft.** Keine Deaktivierung, keine Karenzzeit.
  - **Kein Umweg über Support, E-Mail oder Telefon und keine Weiterleitung ins Web** — Apple wertet das als „nicht in der App löschbar".
  - Erneute Authentifizierung per Passwort und eine destruktive Bestätigung.
  - Eine ausdrückliche Liste dessen, was gelöscht wird; Drittanbieter-Berechtigungen (Instagram/Facebook) auch beim Anbieter widerrufen.
- [ ] **Zweckbeschreibungen der Berechtigungen** — 5.1.1(ii). **Bare-Workflow-Falle:** Liegt das Verzeichnis `ios/` im Repository, wandern die Beschreibungen aus `app.json` **nicht** nach `Info.plist`. Prüfe von Hand.
- [ ] **Frage keine Berechtigungen ab, die du nicht nutzt.** App B wurde nach 5.1.1 abgelehnt, weil `pickImage` vollen Fotobibliothekszugriff anforderte, bevor der Picker öffnete. Der moderne PHPicker liefert ein Foto **ganz ohne Berechtigung** → lösch den Aufruf.
- [ ] **Anklickbare rechtliche Links** — 2.1(a). App C fiel hier durch: Der Satz „mit dem Anlegen eines Kontos akzeptierst du die Bedingungen" war **reiner Text**, und auf dem Anmeldebildschirm fehlten Links ganz, sodass die Prüfung die Bedingungen nicht sehen konnte. Öffne sie in einem **In-App-Browser** (`expo-web-browser`) statt nach Safari zu werfen, und setz sie auch auf den Anmeldebildschirm.
- [ ] **Priming-Bildschirm für Standort** — 5.1.1(iv). Keine suggestive Beschriftung: „Meinen Standort verwenden" ❌ → „Weiter" ✅. Und keine zwei verwirrenden „Überspringen".

### Nutzergenerierte Inhalte (1.2) — Pflicht, sobald Nutzer etwas veröffentlichen können
App A, Build 51, fiel hier durch. Alle vier sind nötig:
- [ ] Ein **sichtbares** Meldeelement. Eine Long-Press-Geste **reicht nicht**: Die Prüfung findet sie nicht. Setz einen sichtbaren „⋯"-Button neben jede Nachricht, jeden Beitrag und Kommentar.
- [ ] Blockieren (in beide Richtungen: Blockierte können dir auch nicht schreiben).
- [ ] Inhaltsfilterung an **allen** Schreib-Endpunkten, nicht nur den offensichtlichen.
- [ ] Zustimmung bei Direktnachrichten: Wer beginnt, kann eine Nachricht senden, bis die Gegenseite annimmt.

### Kaufsignale (3.1.1) — die tückischste Falle bei kostenlosen und B2B-Apps
App C wurde nach dieser Guideline **zweimal** abgelehnt.
- [ ] **Lass keinen Preis, Tarifnamen, Guthabenzähler, keine Paywall, keinen Upgrade-Button und keinen externen Kauflink übrig.** Build 25 versenkten eine Zeile „Intelligence ist in deinem aktuellen Tarif nicht enthalten — mindestens Solo nötig", ein verbliebener Guthabenzähler und ein Label „1 Guthaben pro Marke". Schon ein Tarifname genügt.
- [ ] **Stütz dich nicht allein auf das Argument 3.1.3(f) „Free Stand-alone Apps" — Apple hat es abgelehnt.** Wir haben es bei Build 26 versucht.
- [ ] **Ein öffentlicher Registrierungsbildschirm zerstört ein B2B-Argument.** Genau das war das schwache Glied: Ein „Beitreten"-Bildschirm liest sich für die Prüfung als Selbstbedienungsverkauf an Endkunden und widerspricht direkt „only sold directly by you to organizations" aus 3.1.3(c). Die Lösung in Build 27 war, **den Registrierungsbildschirm ganz zu löschen** — die App bietet nur noch Anmeldung.

### Metadaten
- [ ] **Altersfreigabe** — 2.3.6. Jede App mit einem „Leute kennenlernen / Networking"-Einschlag braucht `matureOrSuggestiveThemes` mindestens auf `INFREQUENT_OR_MILD`. Per API korrigierbar: `PATCH /v1/ageRatingDeclarations/{id}`.
- [ ] **Liefert die URL der Datenschutzerklärung 200?** Unsere erste Produktionseinreichung bei Play wurde allein deshalb abgelehnt, weil die angegebene URL 404 lieferte.

### Was die Prüfung tatsächlich sieht
- [ ] **Funktioniert das Demo-Konto wirklich?** Teste auf einem Gerät. App A wurde nach 2.1 abgelehnt, weil die Anmeldung über das Apple-Watch-Target nie funktioniert hatte: Sie schickte `email`, während das Backend `identifier` las, und bekam 422 zurück. Niemand merkte es, weil das Telefon-Target das richtige Feld sendete.
- [ ] **Sind die Demo-Daten frisch?** In App A lagen 16 eingespielte Events in der Vergangenheit: Die Prüfung hätte eine leere App geöffnet. Halte ein idempotentes Skript bereit, das die Daten nach vorn schiebt.
- [ ] **Sperrt eine Verifizierungsmauer die Prüfung aus?** Sieht eine registrierte, aber unbestätigte Person nichts, wirkt die App geschlossen. Lass Gäste stöbern; verlange Verifizierung nur beim Schreiben.
- [ ] **Veröffentliche keine leeren oder abgeschalteten Module.** Ein abgeschaltetes Feature-Flag in App A hinterließ einen leeren Bereich „Kurse", der zwei 2.1-Ablehnungen nacheinander auslöste (App Completeness, dann Information Needed). Am Ende wurde er entfernt. **Reiche keine halbe Funktion ein — nimm sie raus.**
- [ ] **Du wählst das Prüfgerät nicht aus.** Bei uns kamen Rückmeldungen von einem iPad Air und einer Apple Watch. Teste auch Targets jenseits des Hauptgeräts.

### Plattformintegration (4.0 Design)
- [ ] **Ist deine Karten- oder Standortfunktion in die native App integriert?** App B wurde nach 4.0.0 abgelehnt, weil sie Nutzer nur an Google Maps weiterreichte. Biete Apple Maps (`maps.apple.com`) als Option.

### Android
- [ ] **Passt die Werbe-ID-Erklärung zum Manifest?** Entpacke das `.aab` und suche `com.google.android.gms.permission.AD_ID`. Fehlt es, muss die Erklärung „wird nicht verwendet" lauten: eine falsche Erklärung blockiert das Release.
- [ ] **Vertriebsländer.** Eine unserer Apps war in der Produktion versehentlich auf **ein einziges Land** beschränkt, während iOS in 175 auslieferte.
- [ ] **Hintergrundstandort** kann die Standort-Berechtigungserklärung von Play auslösen.
- [ ] **Maps-API-Schlüssel** in `app.json > android.config.googleMaps.apiKey`: Ohne ihn **stürzt `react-native-maps` auf Android bei der nativen Initialisierung ab**. Auf iOS passiert das nicht (dort ist Apple Maps die Voreinstellung), und genau deshalb rutscht es durch.
- [ ] **Google Sign-In braucht ZWEI SHA-1:** deinen Upload-Schlüssel **und** Plays App-Signaturschlüssel. Play erzeugt seinen erst nach dem ersten AAB-Upload; fehlt diese SHA-1 im Android-OAuth-Client, ist Google Sign-In im Play-Build kaputt. Auf einem Emulator lässt es sich auch nicht testen (die Debug-SHA-1 ist nicht registriert): nötig ist ein von Play signierter Build auf einem Gerät.

---

## Ablehnungskatalog

| Guideline | Was der Store sagte | Tatsächliche Ursache | Lösung |
|---|---|---|---|
| **4.2** Minimum Functionality | „small, or niche, set of users" | **Unser eigener Satz in den Review Notes** | Entdeckungsablauf ohne Konto + öffentliche Endpunkte + Inhaltsbalance |
| **1.2** UGC | keine Filterung / Meldung / Blockierung | Melden gab es nur hinter einer unsichtbaren Long-Press-Geste; in Direktnachrichten und Lobbys gar nicht | Sichtbares „⋯"-Menü auf 8 Oberflächen, Filter an 9 Schreib-Endpunkten, DM-Zustimmung |
| **2.1** Demo-Konto | Anmeldung nicht möglich | Watch-Target schickte `email`, Backend las `identifier` → 422 | Feld korrigiert; Abschnitt „WATCH — PLEASE READ FIRST" in den Notes |
| **2.1** App Completeness | „could not access the courses" | Abgeschaltetes Feature-Flag; der Bereich erschien leer | Funktion vollständig entfernt |
| **2.1** Information Needed | „auf wie viele Nutzer zielt ihr?" | Dasselbe leere Modul plus die 4.2-Rahmung | Notes neu geschrieben |
| **2.3.6** Altersfreigabe | „Mature or Suggestive Themes" | Kennenlern-Thematik nicht deklariert | `PATCH ageRatingDeclarations` über die ASC-API |
| **3.1.1** In-App-Kauf | Kaufsignale in einer kostenlosen App | Tarifname, Guthabenzähler, Text „Solo-Tarif nötig" | Alle Preis- und Tarifspuren entfernt |
| **3.1.1** (zweites Mal) | dieselbe Guideline erneut | 3.1.3(f) reichte nicht; ein **öffentlicher Registrierungsbildschirm** widersprach dem B2B-Argument | Registrierung gelöscht, nur noch Anmeldung |
| **2.1(a)** App Completeness | Bedingungen nicht einsehbar | Rechtstext war reiner, nicht anklickbarer Text und fehlte auf dem Anmeldebildschirm | Anklickbare Links im In-App-Browser |
| **5.1.1(v)** Data Collection | keine Kontolöschung | — | Kontolöschung in der App |
| **5.1.1(ii)** | fehlende Zweckbeschreibung | Bare Workflow synchronisiert `app.json` nicht nach `Info.plist` | `Info.plist` direkt bearbeiten |
| **5.1.1(iv)** Standortablauf | suggestiver Priming-Bildschirm | Buttontexte und doppeltes Überspringen | „Weiter", ein einziger Ausgang |
| **5.1.1** Fotozugriff | forderte Bibliotheksberechtigung | Der PHPicker brauchte diese Berechtigung nie | Berechtigungsaufruf entfernt |
| **4.0.0** Design | „not integrated with built-in mapping" | Nur Weiterreichen an Google Maps | Apple-Maps-Option ergänzt |
| **Play** (Produktion) | Einreichung abgelehnt | Angegebene Datenschutz-URL lieferte 404 | Dauerhafter Alias und korrigierter Konsoleneintrag |

**Hinweis:** Eine Ablehnung zu beheben kann die nächste einladen. Eine App durchlief vier Ablehnungen in Folge, eine andere drei. Geh nach jeder Korrektur die **gesamte** Liste erneut durch.

---

## Fallen der App-Store-Connect-API und der Erklärungen

Gefunden beim Ausfüllen der Einreichungsfelder über die API:

- **Den Screenshot-Typ `APP_IPHONE_69` gibt es nicht.** Der größte iPhone-Typ der API ist `APP_IPHONE_67` (1290×2796). Bilder in 1320×2868 für das 6,9-Zoll-Gerät werden **abgelehnt**: Lade 6,7" hoch und lass Apple skalieren.
- **`whatsNew` lässt sich in einer ersten Version nicht bearbeiten** — 409, „cannot be edited at this time". Es existiert nur für Updates.
- **Die Feldtypen der Altersfreigabe sind gemischt:** teils BOOLEAN (`messagingAndChat`, `userGeneratedContent`, `advertising`), teils STRING-Enums (`contests`, `profanityOrCrudeHumor` → `NONE` / `INFREQUENT_OR_MILD` / `FREQUENT_OR_INTENSE`). Der falsche Typ liefert 409, und die Fehlermeldung nennt die richtige Menge.
- **Apple hat die Altersstufen 2025 geändert: 12+ existiert nicht mehr.** Es sind 4+, 9+, 13+, 16+, 18+. Ehrliche Antworten können 4+ ergeben; heb es mit `ageRatingOverrideV2` an (z. B. `THIRTEEN_PLUS`).

**App-Privacy-Erklärung:**
- **Eine Ausweisnummer ist keine „Sensitive Info".** Apples sensible Kategorie umfasst Herkunft, Religion, sexuelle Orientierung, Biometrie und Ähnliches; eine Ausweisnummer steht nicht darauf → richtiger Eimer ist **„Other Data Types"**.
- **Bankdaten in deiner eigenen Datenbank sind „Collected".** Apple befreit nur, wenn der Zahlungsanbieter sie hält und du keinen Zugriff hast.
- ⚠️ **Blindklick-Falle:** Der Assistent rendert je nach Datentyp unterschiedlich hoch. Wiederholte Klicks auf dieselbe Position erzeugten Antworten wie „Nutzer-ID wird zum Tracking verwendet: JA", die falsch waren. Prüfe per Screenshot den Endzustand jedes Punkts.

---

## Build- und Upload-Fallen

### Der Release-Zug
**Du kannst keinen neuen Build gegen eine bereits genehmigte Versionszeichenkette hochladen** — altool-Fehler 90062 / 90186 („Invalid Pre-Release Train ... closed"). Erhöhe `version` in `app.json` und **bau neu**: Die Versionszeichenkette steckt im IPA. Uns kostete das einen ganzen Build.

### Upload
- `eas submit` kann hängen bleiben (über 23 Minuten ohne Ausgabe) oder mit „Failed to authenticate for session" scheitern. **Der verlässliche Weg ist altool direkt:**
  ```bash
  xcrun altool --upload-app -f build.ipa -t ios --apiKey <KEY_ID> --apiIssuer <ISSUER_ID>
  ```
  Leg die `.p8` nach `~/.appstoreconnect/private_keys/AuthKey_<KEY_ID>.p8`. Dauert etwa 15 Sekunden.
- **„Upload succeeded" ist nicht „angenommen".** Apple kann während der Verarbeitung noch ablehnen. Frage bis `VALID` ab und setz dann den vorherigen Build auf abgelaufen (`PATCH /v1/builds/{id}` mit `{"expired": true}`).
- **Watch- und Widget-Targets brauchen ein Icon** (`CFBundleIconName`), sonst weist Apple den Upload mit Fehler **90713** ab.
- **ITMS-90062** heißt „diese Version ist bereits veröffentlicht": Versionszeichenkette anheben.
- **ITMS-90863** (Apple-Silicon-Symbolwarnung) ist **bei Expo-Apps normal und führt zu keiner Ablehnung.** Nicht hinterherjagen.

### Reihenfolge beim erneuten Einreichen
1. **Zwei Versionen können nicht gleichzeitig in Prüfung sein.** Storniere die bestehende `reviewSubmission` (`canceled=true`) und warte auf CANCELING → COMPLETE.
2. Die Version wird `DEVELOPER_REJECTED` (bearbeitbar) → PATCH der Versionszeichenkette → PATCH der Build-Beziehung.
3. ⚠️ **Vertauschungsfalle:** Direkt nach dem Stornieren gibt das Anhängen 409 zurück, und läuft das Skript weiter, reicht es den **alten** Build ein. Wiederhole das Anhängen und **prüfe den angehängten Build vor dem Einreichen** (`GET /appStoreVersions/{id}/build`).
4. ⚠️ `POST reviewSubmissionItems` kann 409 `ENTITY_STATE_INVALID` liefern, während der Übergang läuft. Sekunden später klappt es: wiederholbar machen.

### Lokale Build-Umgebung
- **Aktualisiert sich Xcode mitten in der Sitzung**, scheitern Builds mit „iOS X Platform Not Installed". Lösung: `xcodebuild -downloadPlatform iOS` (~8,5 GB, kein sudo) und `xcodebuild -runFirstLaunch`. Dass es am selben Morgen noch lief, beweist nichts über den jetzigen Zustand.
- **CocoaPods unter Ruby 4.0:** `pod install` stirbt mit `Unicode Normalization not appropriate for ASCII-8BIT`. Führ es mit `LANG=en_US.UTF-8 LC_ALL=en_US.UTF-8` aus.
- **Modular Headers im Podfile:** GoogleSignIn 9.2.0 braucht `:modular_headers => true` für `AppCheckCore`, `GoogleUtilities` und `RecaptchaInterop`.
- **Ein Provisioning Profile, das älter ist als eine neue Capability**, lässt den lokalen Build scheitern. Nicht-interaktives EAS aktualisiert keine Anmeldedaten: entweder interaktiv erneuern oder über die ASC-API gehen.
- **Apple-Developer-Capabilities lassen sich per API aktivieren** (ohne Portal): `POST /v1/bundleIdCapabilities`. Ohne `settings`-Objekt gibt es 409.
- **`ANDROID_HOME` ist Pflicht** bei lokalen Android-Builds, sonst meldet Gradle „SDK location not found".
- **Bearbeite nie Quelldateien, während ein Archive kompiliert** — Metro bettet ein halb geschriebenes Bundle ein und die App stürzt beim Start ab.
- **EAS-Temporärdateien wachsen unbegrenzt** (bei uns bis 35 GB). Räum auf; eine volle Platte lässt den Build mit „No space left" scheitern.
- Springende Build-Nummern nach fehlgeschlagenen Versuchen sind normal.

### Play-Release-Fehler
- **„Dieses Release wird bestehenden Nutzern nicht zur Verfügung gestellt…"** → Version Code erhöhen oder erst über internen/geschlossenen Test veröffentlichen.
- **„Dieses Release fügt keine App-Bundles hinzu und entfernt keine."** → Das AAB wurde nicht sauber hochgeladen; Version Code prüfen und erneut hochladen.
- **Native Debug-Symbole** gehören in eine `native-debug-symbols.zip` mit ABI-Verzeichnissen (`armeabi-v7a/`, `arm64-v8a/`, `x86_64/`, jeweils mit `libapp.so`) und ohne `__MACOSX`- oder `.DS_Store`-Einträge.
- ⚠️ **Fristen für das Target-API-Level.** Play blockiert Update-Veröffentlichungen für Apps, die die Frist verpassen. Behalte das Datum im Blick.
- **Die AD_ID-Feinheit:** Firebase Analytics verlangt die Berechtigung im Manifest und eine „wird verwendet"-Erklärung; eine App ohne Werbung braucht beides nicht. **Die Regel: Die Erklärung muss exakt zum Manifest passen** — eine Abweichung in beide Richtungen blockiert das Release.

### Abstürze, die es nur in Standalone-Builds gibt
- **Simulatoren und Dev Clients fangen sie nicht.** Teste auf einem echten Gerät per Kabel mit `devicectl --console`.
- Steht `.env` in `.gitignore`, erreicht sie das EAS-Archiv nie: leere Variablen im Bundle und Absturz beim Start. In einer App stürzten *alle* Builds deswegen ab.
- Ein dynamisch importiertes natives Modul, das nicht installiert ist, bleibt in der Entwicklung unsichtbar (Metro liefert es aus) und stürzt im Standalone-Build mit `RCTFatalException: Cannot find module` ab.
- **Hermes speichert Strings als UTF-16.** Nicht-ASCII-Strings im Bundle als UTF-8 zu suchen liefert nichts: prüfe in UTF-16.

---

## Store-Registrierung — einmalig und manuell

- **Der App-Eintrag in App Store Connect lässt sich nicht per API anlegen.** Wir haben es versucht und bestätigt. Mach es im Browser.
- **Auch das Anlegen der App in der Play Console ist beim ersten Mal manuell.**
- **Die Bundle-ID bindet sich dauerhaft an den Eintrag und lässt sich nicht ändern.**
- **„Kostenlos" bei Play ist unumkehrbar**: Nach der Veröffentlichung geht kein Wechsel auf kostenpflichtig.
- ⚠️ **Nicht-ASCII-Zeichen können bei der Registrierung verloren gehen.** In einem Apple-Einzelkonto ist der im App Store gezeigte Entwicklername dein rechtlicher Name; unserer verlor bei der Registrierung seine Diakritika. Die Korrektur über App Store Connect → Business → Legal Entity **funktioniert nicht**: Dieser Ablauf zieht dich in Adressprüfung und die Kette des Paid Apps Agreement, und der Name allein wird nicht gespeichert. Der funktionierende Weg ist Apple Support → „Membership & Account" → Korrektur des rechtlichen Namens, mit Identitätsprüfung. **Prüfe die Schreibweise bei der Registrierung Zeichen für Zeichen.**

## Grenzen für einen KI-Assistenten, der diesen Leitfaden ausführt

- **Gib niemals das Apple- oder Google-Passwort und den 2FA-Code ein.** App Store Connect verlangt eine eigene Anmeldung (die Sitzung des Entwicklerportals wird nicht übernommen). Der Ablauf: Die Person meldet sich selbst an, bestätigt, und der Assistent übernimmt ab da die API- und Konsolenschritte.
- **Datei-Uploads über den Browser sind auf 10 MB begrenzt;** ein typisches `.aab` ist über 60 MB groß. Lass die Person hochladen oder automatisiere es mit einem Play-Dienstkonto und `eas.json > submit.android`.
- **Setze niemals Häkchen bei Erklärungen oder Einwilligungen** ohne ausdrückliche Zustimmung der Person.

---

Kommt eine neue Ablehnung, finde zuerst die Ursache und ergänze dann hier eine Zeile. Ein Leitfaden ist so viel wert wie das, was ihm der letzte Vorfall beigebracht hat.
