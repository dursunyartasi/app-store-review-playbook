# App-Store-Einreichung

Der Build ist hochgeladen und `VALID`. Jetzt kommen Metadaten, Erklärungen und Review Notes — dort wurden die meisten unserer Ablehnungen tatsächlich entschieden.

## Den Eintrag anlegen (einmalig)

**Der App-Eintrag in App Store Connect lässt sich nicht über die API anlegen.** Wir haben es versucht und bestätigt; mach es im Browser. Die Bundle-ID bindet sich dauerhaft an diesen Eintrag.

Alles danach — Build anhängen, Metadaten, Altersfreigabe, Review Notes, Einreichung — lässt sich über die ASC-API steuern.

## Screenshots

- **Den Screenshot-Typ `APP_IPHONE_69` gibt es nicht.** Der größte iPhone-Typ, den die API akzeptiert, ist `APP_IPHONE_67` (1290×2796). Bilder in 1320×2868 für das 6,9-Zoll-Gerät werden **abgelehnt**. Lade 6,7" hoch und lass Apple hochskalieren.
- `whatsNew` **lässt sich in einer ersten Version nicht bearbeiten** — 409, „cannot be edited at this time". Es existiert nur für Updates.

## Altersfreigabe

- Die Feldtypen sind gemischt: teils BOOLEAN (`messagingAndChat`, `userGeneratedContent`, `advertising`), teils STRING-Enums (`contests`, `profanityOrCrudeHumor` → `NONE` / `INFREQUENT_OR_MILD` / `FREQUENT_OR_INTENSE`). Der falsche Typ liefert 409, und die Fehlermeldung nennt die richtige Menge.
- **Apple hat die Altersstufen 2025 geändert: 12+ gibt es nicht mehr.** Es sind 4+, 9+, 13+, 16+, 18+.
- Ehrliche Antworten können 4+ ergeben; heb es mit `ageRatingOverrideV2` an (z. B. `THIRTEEN_PLUS`).
- **Hat die App irgendeinen „Leute kennenlernen / Networking"-Einschlag, deklariere `matureOrSuggestiveThemes` mindestens als `INFREQUENT_OR_MILD`.** Es auf keine zu lassen war eine 2.3.6-Ablehnung.

## App-Privacy-Erklärung

- **Eine nationale Ausweisnummer ist keine „Sensitive Info".** Apples sensible Kategorie umfasst Herkunft, Religion, sexuelle Orientierung, Biometrie und Ähnliches; eine Ausweisnummer steht nicht darauf, also ist **„Other Data Types"** der richtige Eimer.
- **Bankdaten, die du selbst speicherst, gelten als „Collected".** Apple befreit dich nur, wenn der Zahlungsanbieter sie hält und du keinen Zugriff hast.
- ⚠️ **Klick dich nicht blind durch den Assistenten.** Er rendert je nach Datentyp unterschiedlich hoch, deshalb erzeugten wiederholte Klicks auf dieselbe Position Antworten wie „Nutzer-ID wird zum Tracking verwendet: JA", die schlicht falsch waren. Mach Screenshots und prüfe den Endzustand jedes Punkts.

## Review Notes — der Text mit der größten Hebelwirkung

Eine unserer Ablehnungen kam vollständig aus diesem Feld. Apples 4.2-Ablehnung „small, or niche, set of users" zitierte uns unseren eigenen Satz zurück:

> "We are targeting a deliberately **small**, curated early community — **a few dozen** invited members, **not a mass-market** social network."

**Beschreibe die App niemals als klein, Nische, nur auf Einladung, geschlossen, privat, für eine bestimmte Community oder nicht massentauglich.** Apple liest das als Ad-Hoc-Verteilung, nicht als App Store.

Schreib stattdessen: offen für alle, kostenlos, überall herunterladbar; jede kuratierte oder Mitgliedschaftsebene ist *optional*. Beschreibe dann in nummerierten Schritten den Weg, den eine prüfende Person **ohne Konto** gehen kann. Existiert dieser Weg in der App nicht wirklich, bau ihn vor der Einreichung — genau das hat unsere 4.2 gelöst.

**Das Feld ist auf 4000 Zeichen begrenzt.** Darüber gibt es 409.

Hat die App ein ungewöhnliches Ziel (Watch, Widget, ein gerätespezifischer Ablauf), setz ganz oben einen Abschnitt „PLEASE READ FIRST" mit expliziten Anmeldeschritten.

## Demo-Konto

Setze „Sign-In Required" und hinterlege die Zugangsdaten.

- **Teste sie vorher auf einem Gerät.** Eine 2.1-Ablehnung kam von einer Anmeldung, die auf dem Watch-Target nie funktioniert hatte.
- **Sorg dafür, dass das Konto Inhalte hat.** In einer App lagen 16 von 17 eingespielten Events in der Vergangenheit, die prüfende Person hätte eine leere App geöffnet. Halte ein idempotentes Skript bereit, das Demo-Daten nach vorn schiebt, und lass es vor jeder Einreichung laufen.
- **Eine Verifizierungsmauer sperrt die Prüfung aus.** Sieht eine registrierte, aber unbestätigte Person nichts, wirkt die App geschlossen. Lass Gäste stöbern und verlange Verifizierung nur bei Schreibaktionen.
- **Schließe das Demo-Konto nach der Freigabe.** Sein Passwort liegt in App Store Connect.

## Rechtliche Links

Die Links zu Nutzungsbedingungen und Datenschutz müssen **anklickbar** sein, in einem In-App-Browser öffnen statt nach Safari zu werfen, und auch auf dem **Anmeldebildschirm** erscheinen, nicht nur bei der Registrierung. Nicht anklickbarer Text war eine 2.1(a)-Ablehnung: Die prüfende Person konnte die Bedingungen nicht lesen und lehnte allein deshalb ab.

## Wenn die App kostenlos ist, aber irgendwo etwas verkauft

3.1.1 ist die Falle für kostenlose und B2B-Apps. **Entferne jeden Preis, Tarifnamen, Guthabenzähler, jede Paywall, jeden Upgrade-Button und jeden externen Kauflink.** Ein einzelner Tarifname reichte, um einen Build zu versenken.

Das Argument 3.1.3(f) „Free Stand-alone Apps" **funktionierte bei uns allein nicht.** Das schwache Glied war ein öffentlicher Registrierungsbildschirm: Er liest sich als Selbstbedienungsverkauf an Endkunden und widerspricht der Formulierung „only sold directly by you to organizations" in 3.1.3(c). Wir löschten die Registrierung und veröffentlichten nur mit Anmeldung.

## Einreichen und nach einer Ablehnung erneut einreichen

Die Reihenfolge zählt. Ein Fehler reicht stillschweigend das falsche Binary ein.

1. **Zwei Versionen können nicht gleichzeitig in Prüfung sein.** Storniere die bestehende `reviewSubmission` (`canceled=true`) und warte auf CANCELING → COMPLETE.
2. Die Version wird `DEVELOPER_REJECTED` und bearbeitbar. PATCH die Versionszeichenkette, dann die Build-Beziehung.
3. ⚠️ **Die Vertauschungsfalle.** Direkt nach dem Stornieren gibt der Anhäng-Aufruf 409 zurück. Läuft dein Skript trotzdem weiter, reicht es den **alten** Build ein. Wiederhole das Anhängen und **prüfe** vor dem Einreichen mit `GET /appStoreVersions/{id}/build`. Einmal haben wir so den falschen Build veröffentlicht.
4. ⚠️ `POST reviewSubmissionItems` kann 409 `ENTITY_STATE_INVALID` liefern, während der Zustandswechsel läuft. Sekunden später klappt es: mach den Schritt wiederholbar.

Der Veröffentlichungstyp ist standardmäßig **manuell**: Nach der Freigabe muss jemand noch auf Veröffentlichen drücken.

## Rechne mit mehr als einer Runde

Eine App durchlief vier Ablehnungen in Folge, eine andere drei. Eine zu beheben kann die nächste freilegen, und eine Korrektur an einer Stelle kann anderswo ein Problem schaffen. **Lies nach jeder Korrektur die gesamte Liste in `05-ablehnungen.md`**, nicht nur den Punkt, den du geändert hast.
