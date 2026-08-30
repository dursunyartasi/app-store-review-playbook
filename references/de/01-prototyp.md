# Von null zu einem laufenden Prototypen

Ziel: eine App, die auf dem Handy der Person läuft und so gebaut ist, dass die Store-Anforderungen aus `05-ablehnungen.md` bereits erfüllt statt nachträglich angeflickt sind.

## Entscheidungen vor der ersten Datei

**Managed oder Bare Workflow.** Managed hält `ios/` und `android/` generiert; Bare legt sie ins Repository. Bare gibt dir native Kontrolle und nimmt dir das hier: **`app.json` ist nicht mehr die Quelle der Wahrheit.** Die Zweckbeschreibungen der Berechtigungen wandern nicht mehr nach `Info.plist`, und die Version kommt ab dann aus `CFBundleShortVersionString` in `Info.plist` plus `MARKETING_VERSION` im pbxproj. Beides hat uns Ablehnungen gekostet. Fang mit Managed an, außer ein natives Modul zwingt dich.

**Bundle Identifier.** Wähle ihn jetzt und mit Bedacht — sobald ein App-Store-Connect-Eintrag existiert, ist **die Bundle-ID dauerhaft**. Nutze Reverse-DNS auf einer Domain, die dir gehört.

**Der Name, den Apple anzeigt.** In einem Apple-Developer-Einzelkonto ist der im App Store angezeigte Entwicklername dein rechtlicher Name, so wie er registriert wurde. Nicht-ASCII-Zeichen können bei der Registrierung stillschweigend verloren gehen (bei uns fielen die türkischen Diakritika weg), und die Selbstkorrektur über App Store Connect **funktioniert nicht**: Dieser Ablauf zieht dich in Adressprüfung und die Kette des Paid Apps Agreement, und der Name allein wird nie gespeichert. Die Korrektur erfordert eine Support-Anfrage mit Identitätsprüfung. **Prüfe die Schreibweise bei der Registrierung Zeichen für Zeichen.**

## Grundgerüst

```bash
npx create-expo-app@latest my-app
cd my-app
npx expo start            # QR-Code mit Expo Go scannen
```

Expo Go reicht, bis du ein natives Modul hinzufügst oder einen signierten Build brauchst. Danach brauchst du einen Development Build oder ein echtes Archive.

## Bau die Store-Anforderungen jetzt ein

Am ersten Tag sind sie billig, nach einer Ablehnung teuer.

**Wenn Nutzer sich anmelden können — Kontolöschung (5.1.1(v)).** Sie muss innerhalb der App erreichbar sein, sofort und dauerhaft. Keine Deaktivierung, keine Wartefrist, kein „schreib an den Support, um zu löschen", keine Weiterleitung auf eine Website. Frage das Passwort erneut ab, zeige eine destruktive Bestätigung, liste auf, was gelöscht wird, und widerrufe Drittanbieter-Berechtigungen auch auf der Seite des Anbieters.

**Wenn Nutzer etwas veröffentlichen können — die vier Anforderungen aus 1.2.** Ein sichtbares Element an jeder Nachricht, jedem Beitrag und Kommentar (ein „⋯"-Button; die Long-Press-Geste ist für die Prüfung unsichtbar und wurde abgelehnt), Blockieren, das in beide Richtungen wirkt, Inhaltsfilterung an **allen** Schreib-Endpunkten, und ein Zustimmungsschritt, bevor eine fremde Person mehr als eine Direktnachricht senden kann.

**Rechtliche Links müssen anklickbar sein (2.1(a)).** „Mit der Registrierung akzeptierst du die Nutzungsbedingungen" als reiner Text ist eine Ablehnung. Mach echte Links daraus, öffne sie in einem In-App-Browser statt die Person nach Safari zu werfen, und setze sie auch auf den Anmeldebildschirm, nicht nur auf die Registrierung.

**Berechtigungen.** Frage nichts ab, was du nicht nutzt. Vollen Fotobibliothekszugriff vor dem Öffnen des Pickers anzufordern war eine Ablehnung: Der moderne iOS-Picker liefert ein Foto ganz ohne Berechtigung. Priming-Bildschirme dürfen keine suggestive Buttonbeschriftung haben: „Weiter", nicht „Meinen Standort verwenden".

**Eine Kontaktadresse, die wirklich E-Mails empfängt.** Wenn du eine im Store-Eintrag oder in den In-App-Regeln veröffentlichst, braucht die Domain einen MX-Eintrag. Siehe die MX-Falle in `06-infrastruktur.md` — unsere konnte senden, aber nicht empfangen, sodass die Moderationsadresse aus unseren veröffentlichten Regeln niemanden erreichte.

## Umgebungsvariablen

```
.env            → wie üblich in .gitignore
.easignore      → DAS hier liest EAS, und es ersetzt .gitignore
```

**Eine ignorierte `.env` erreicht das EAS-Archiv nie.** Das Bundle geht mit leeren Variablen raus und die App stürzt beim Start ab — **nur** in Standalone-Builds, weshalb Simulator und Dev Client tadellos aussehen. In einer unserer Apps stürzten ausnahmslos alle Builds daran ab, bis wir es fanden. Konfiguriere entweder EAS-Umgebungsvariablen oder stelle sicher, dass `.easignore` nicht ausschließt, was der Build braucht.

## Bring es auf ein echtes Gerät

Ein Simulator beweist nicht, dass die App funktioniert. Abstürze, die nur im Standalone-Build auftreten, sind genau die Sorte, die bei der Prüfung ankommt:

```bash
npx expo export --platform ios --output-dir /tmp/exportcheck   # findet Import-Fehler früh
```

Dann per Kabel bauen und installieren und das Log mit `devicectl --console` verfolgen. Ein dynamisch per `import()` geladenes natives Modul, das nicht installiert ist, bleibt in der Entwicklung unsichtbar — Metro liefert es aus — und stürzt im Standalone-Build mit `RCTFatalException: Cannot find module` ab.

## Bevor du weitergehst

Lass `npx tsc --noEmit` und deine Tests laufen und mach sie sauber. Ab hier kostet jeder Build-Zyklus 5 bis 40 Minuten und, sobald die Prüfung läuft, Tage.

Weiter: `02-testflight-ios.md` oder `03-google-play.md`.
