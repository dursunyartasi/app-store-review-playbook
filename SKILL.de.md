---
name: mobile-app-shipping-de
description: Vollständiger Leitfaden zum Bauen und Veröffentlichen einer mobilen App im App Store und bei Google Play, geschrieben aus echten Ablehnungen und echten Build-Fehlern. Verwende ihn, wenn jemand einen mobilen Prototypen starten, Expo/React Native einrichten, ein IPA oder AAB bauen, zu TestFlight hochladen, zu Google Play hochladen, zur App-Store-Prüfung einreichen, Review Notes schreiben, Store-Erklärungen ausfüllen oder eine Ablehnung beheben will. Löst außerdem aus bei - "App abgelehnt", "Guideline", "App Review", "TestFlight", "eas build", "altool", "Provisioning Profile", "AAB", "Play Console", "App Store Connect", "Altersfreigabe", "Datenschutzerklärung", "Screenshots".
metadata:
  version: 2.0.0
  quelle: 8 veröffentlichte iOS-/Android-Apps, 2026
---

# Eine mobile App veröffentlichen

Alles hier stammt aus Apps, die tatsächlich in den Stores gelandet sind, und aus den Ablehnungen und Build-Fehlern auf dem Weg dorthin. Nichts davon ist aus der offiziellen Dokumentation umformuliert.

## Finde zuerst heraus, wo die Person steht

Frage nur, was du noch nicht weißt. Wenn die Nachricht eine Frage bereits beantwortet, überspring sie. **Stelle nie mehr als fünf Fragen.**

1. **Was willst du gerade tun?**
   `neuer Prototyp` · `bauen und aufs Handy bringen` · `TestFlight` · `Google Play` · `zur Prüfung einreichen` · `ich wurde abgelehnt`
2. **Welche Plattformen?** iOS, Android oder beide.
3. **Melden sich Nutzer an oder erstellen sie Inhalte?** (Konten, Beiträge, Kommentare, Nachrichten, Fotos)
4. **Hast du die Entwicklerkonten?** Apple Developer kostet 99 $ im Jahr und ist Pflicht, bevor überhaupt etwas auf ein echtes Gerät kommt. Google Play kostet einmalig 25 $.
5. **Gibt es ein Backend, oder wird eines gebraucht?**

Dann geh zur passenden Datei. Lies nur diese eine.

| Antwort | Lies |
|---|---|
| neuer Prototyp | `references/de/01-prototyp.md` |
| bauen / TestFlight | `references/de/02-testflight-ios.md` |
| Google Play | `references/de/03-google-play.md` |
| zur Prüfung einreichen | `references/de/04-app-store-einreichung.md` |
| ich wurde abgelehnt | `references/de/05-ablehnungen.md` |
| Backend, Datenbank, E-Mail | `references/de/06-infrastruktur.md` |

## Was die Antworten ändern

**Frage 3 wiegt am schwersten.** Wenn Nutzer sich anmelden können, schuldest du Apple eine Kontolöschung in der App (5.1.1(v)), sonst wirst du abgelehnt. Wenn Nutzer etwas veröffentlichen können, das andere sehen, schuldest du die vier Punkte aus 1.2: sichtbares Melden, Blockieren, Inhaltsfilterung und Zustimmung bei Direktnachrichten. Das sind vier getrennte Dinge, und eine Long-Press-Geste gilt nicht als sichtbar. Bau sie in den Prototypen ein. Nach einer Ablehnung nachzurüsten kostet einen vollen Prüfzyklus, also Tage.

**Frage 4 ist das Tor zu allem anderen.** Ohne bezahltes Apple-Konto gibt es kein TestFlight, keine Geräteinstallation über ein kostenloses 7-Tage-Profil hinaus und keine Einreichung. Sag das, bevor jemand einen ganzen Tag mit Bauen verbringt.

**Frage 5 hat eine günstige Antwort.** `references/de/06-infrastruktur.md` beschreibt ein selbst gehostetes Setup, das Preise pro Dienst vermeidet: Coolify auf einem gewöhnlichen VPS, PostgreSQL und der kostenlose Tarif von Brevo für transaktionale E-Mails.

## Regeln, die in jeder Phase gelten

- **Gib niemals das Apple- oder Google-Passwort und den 2FA-Code der anderen Person ein.** App Store Connect verlangt eine eigene Anmeldung, die Sitzung aus dem Entwicklerportal wird nicht übernommen. Bitte die Person, sich anzumelden, warte auf die Bestätigung und übernimm dann die API- und Konsolenschritte.
- **Setze keine Häkchen bei Erklärungen oder Einwilligungen für sie.** Das sind rechtliche Aussagen über ihre App.
- **„Upload succeeded" heißt nicht „angenommen".** Apple lehnt auch während der Verarbeitung ab. Frage ab, bis der Build `VALID` ist.
- **Teste, was die prüfende Person sehen wird, nicht was du siehst.** Die meisten Ablehnungen in `05-ablehnungen.md` funktionierten auf dem Gerät und im Konto der Entwicklerin oder des Entwicklers einwandfrei.
- **Bevor du der Prüfung die Schuld gibst: prüfe, ob das Unerreichbare von außerhalb deiner Maschine wirklich erreichbar ist.** Mehrere Ablehnungen mit Guideline-Nummer waren in Wahrheit ein 404, ein abgeschaltetes Feature-Flag oder ein fehlender DNS-Eintrag.

## Reihenfolge, die nicht getauscht werden darf

Einige Schritte lassen sich nicht umstellen, und ein Fehler kostet ganze Builds:

1. Erhöhe die **Versionszeichenkette** vor dem Bauen, wenn die aktuelle bereits genehmigt oder veröffentlicht ist: ihr Release-Zug ist geschlossen, der Upload wird abgelehnt.
2. Prüfe **vor** dem Bauen, dass die Build-Nummer frei ist, nicht danach.
3. Storniere eine **laufende Einreichung**, bevor du einen neuen Build anhängst. Zwei Versionen können nicht gleichzeitig in Prüfung sein.
4. **Prüfe den angehängten Build** vor dem Einreichen. Nach einer Stornierung kann der Anhäng-Aufruf fehlschlagen, während der restliche Ablauf weiterläuft und das alte Binary einreicht.
