# Die Infrastruktur hinter dem Leitfaden

Der [Ablehnungskatalog](05-ablehnungen.md) handelt davon, die Prüfung zu bestehen. Diese Datei handelt davon, was darunter läuft: die Infrastruktur, auf der diese Apps veröffentlicht werden, und die Ausfälle, die uns beim Betrieb begegnet sind.

Dieselbe Regel wie im Leitfaden: nur Dinge, die wirklich passiert sind. Das ist **was wir nutzen**, keine Behauptung darüber, was du nutzen solltest.

---

## Was bei uns läuft

| Schicht | Wahl | Warum |
|---|---|---|
| Mobil | **Expo / React Native** (SDK 57) | Eine Codebasis für beide Stores; EAS oder vollständig lokale Builds |
| Web / API | **Next.js** | Dasselbe TypeScript auf beiden Seiten |
| Datenbank | **PostgreSQL**, selbst gehostet | Vorhersehbare Kosten, keine Überraschungen bei Preisen pro Zeile |
| ORM | **Prisma** | Migrationen, die wir prüfen können, bevor sie die Produktion berühren |
| Dateien | **MinIO** (S3-kompatibel) oder Cloudflare R2 | Selbst gehostete Objekte, keine Rechnung für ausgehenden Traffic |
| Hosting | **Coolify** auf einem gewöhnlichen VPS | Selbst gehostetes PaaS: Git-Deploys, TLS und Container ohne Preis pro Dienst |
| E-Mail | **Brevo**, kostenloser Tarif über SMTP | 300 Mails pro Tag gratis, lange genug für OTP und Benachrichtigungen |
| Zahlungen (Türkei) | **iyzico** | Lokale Karten und Ratenzahlung, was Stripe dort nicht abdeckt |

Eine App läuft auf Supabase statt auf selbst gehostetem PostgreSQL. Beides funktioniert; unten ist markiert, welche Lektionen Supabase-spezifisch sind.

---

## Coolify und Deployment

Coolify ist ein selbst gehostetes PaaS auf deinem eigenen VPS. Es nimmt dir die Hosting-Rechnung pro Dienst und übergibt dir die Betriebsausfälle, die eine Managed-Plattform abgefangen hätte.

### Speicherdruck ist der Ausfall, den du wirklich erlebst

Sobald die Serverfestplatte etwa **80 %** überschreitet, scheitern Deployments beim Export der Layer, obwohl der Build selbst erfolgreich war. Coolify zeigt das als `exit code 255` oder generische `DeploymentException` — **die echte Ursache bleibt verborgen.** Der Export braucht etwa 20 GB frei.

```bash
docker system df           # erst nachsehen
docker builder prune -af   # der Build-Cache ist das, was sich gefahrlos löschen lässt
```

Aufräumen ist **nicht symmetrisch**, und es falsch herum zu machen kostet ein
Deployment:

- `docker image prune -af` — jederzeit sicher.
- `docker builder prune -af` — **nie direkt vor einem Deployment.** Es löscht
  den Build-Cache, die `apt-get`-Schicht läuft erneut und lädt Pakete neu; ein
  einziger Netzwerkaussetzer killt dann den Build. Abseits des Deployment-Pfads
  ist es das Wirksamste, was du hast.

Images sind meist referenziert, ihr Aufräumen bringt wenig. Weitere
Coolify-Fehlermodi, darunter drei verschiedene Ursachen mit identischem
Fehlerbild, im [Coolify-Betriebshandbuch](https://github.com/dursunyartasi/coolify-operations-playbook). **Fass die Volumes nicht an — das sind deine Anwendungsdaten.** Bei einem Vorfall brachte das die Platte von 92 % auf 83 % und schaffte 7,6 GB Luft; der Deploy lief beim erneuten Versuch durch.

Derselbe Speicherdruck zeigt sich auch als vorübergehendes `No such container: <uuid>`, wenn ein Hilfscontainer mitten im Build stirbt. Speicherknappheit erzeugt dasselbe Symptom, also prüfe beides.

### Weiteres Deploy-Verhalten, das man kennen sollte
- **Ein Deploy erzeugt alle Dienste der Compose-Datei neu**, nicht nur den geänderten — einschließlich deines Datenbankcontainers, dessen **Name sich ändert**. Alles, was an einem Containernamen hängt, bricht: nach jedem Deploy neu auflösen.
- **Ein Deploy dauert etwa 200 bis 300 Sekunden.** Frage ab, bis der neue Container und ein HTTP 200 da sind; leite den Erfolg nicht aus dem auslösenden Aufruf ab.
- **Der erste Versuch kann grundlos scheitern** in der Compose-Phase. Ein erneuter Versuch hilft meistens, und die Produktion bleibt oben.
- **Deploys werden standardmäßig nicht per Webhook ausgelöst** — das ist eine manuelle oder API-Aktion.
- Steht dein VPS **hinter Cloudflare**, beachte, dass der Standard-User-Agent von `urllib` blockiert wird. Nutze curl oder setze einen Browser-User-Agent, wenn du gegen deine eigene API skriptest.

### Postgres-Notizen
- **Supabase / PostgREST:** Eine neue Tabelle liefert `PGRST205 "Could not find the table in schema cache"`, obwohl sie existiert. Der REST-Cache ist veraltet. Lösung: `NOTIFY pgrst, 'reload schema'`.
- **Realtime braucht `wal_level=logical`.** Beim Standardwert `replica` abonniert `postgres_changes` fröhlich und liefert dann nie ein Event — ein stiller Ausfall, der wie ein Client-Bug aussieht. Die Umstellung erfordert einen Containerneustart, also nimm ein Wartungsfenster.

---

## E-Mail im kostenlosen Tarif — und die DNS-Falle, die fast alles zerstört hätte

Brevos kostenloser Tarif (300 Mails pro Tag) deckt OTP, Passwort-Resets und Benachrichtigungen lange ab. Richte deine App auf `smtp-relay.brevo.com:587`.

Damit Mails zugestellt statt aussortiert werden, muss die Domain in Brevo als **Authenticated** erscheinen, wofür nötig sind:
- **DKIM** — die beiden CNAME-Einträge von Brevo
- **DMARC** — beginne mit `p=none`
- **SPF** — `include:spf.brevo.com`
- Brevos TXT-Eintrag zur Verifizierung

### ⚠️ Die SPF-Falle
Wir schalteten Cloudflare Email Routing ein, um auf derselben Domain Mails zu *empfangen*. Cloudflare bot an, „die fehlenden Einträge zu ergänzen", sah, dass bereits ein SPF-Eintrag für Brevo existierte, und schlug vor, den Konflikt durch **Löschen des Brevo-Eintrags** zu lösen.

Das anzunehmen hätte jeder von der App versendeten Mail die Authentifizierung entzogen — OTP, Benachrichtigungen, Passwort-Resets — und sie in den Spam geschickt. Richtig ist, beide Includes in **einen** Eintrag zu verschmelzen:

```
v=spf1 include:spf.brevo.com include:_spf.mx.cloudflare.net ~all
```

**Eine Domain muss genau einen SPF-Eintrag haben.** Mehr als einer verletzt den RFC und zerstört den gesamten Versand. Prüfe mit `dig`, vertrau dem Panel nicht.

### Die MX-Falle — und warum sie ein Store-Problem ist
Dieselbe Domain hatte **überhaupt keinen MX-Eintrag**. Sie konnte senden, aber nicht empfangen. Die von uns veröffentlichte Moderations-Kontaktadresse erreichte niemanden.

Das ist nicht bloß ein Mail-Bug. **Guideline 1.2** des App Store erwartet einen funktionierenden Weg, Inhalte zu melden, und unsere eigenen Regeln versprachen eine Antwort binnen drei Werktagen. Eine Adresse, die Mails still verwirft, ist ein gebrochenes Versprechen und ein Prüfungsrisiko. **Wenn du irgendwo im Store-Eintrag oder in den In-App-Regeln eine Kontaktadresse veröffentlichst, schick ihr eine Testnachricht.**

Außerdem: Brevo kann den Versand auf eine IP-Freigabeliste beschränken. Trag sowohl deinen Entwicklungsrechner als auch deinen Server ein, sonst stirbt die Produktionsmail, während lokale Tests durchlaufen.

---

## Notizen zum mobilen Build

Die vollständigen Build- und Upload-Fallen stehen im [Katalog](05-ablehnungen.md). Die Infrastrukturentscheidungen dahinter:

- **Lokale Builds schlagen EAS Remote, solange du iterierst.** Remote-Warteschlangen füllen sich, und nicht-interaktives EAS aktualisiert keine Anmeldedaten — ein Provisioning Profile, das älter als eine neue Capability ist, blockiert dich ausweglos. Lokales `xcodebuild` plus `xcrun altool` ist der Notausgang.
- **Denk `.env` aus der Perspektive von EAS.** Eine ignorierte `.env` erreicht das Archiv nie, was leere Variablen und einen Startabsturz ergibt, der nur in Standalone-Builds auftritt.
- **Lokale Android-Builds brauchen `ANDROID_HOME`**, sonst meldet Gradle „SDK location not found".
- **Automatisiere den Play-Upload mit einem Dienstkonto** (`eas.json > submit.android`). Das `.aab` von Hand hochzuladen ist der Schritt, der am längsten manuell bleibt, und Browser-Automatisierung hilft nicht: Die Dateien liegen weit über jedem Upload-Limit.

---

## Woher der VPS kommt

Coolify braucht einen gewöhnlichen VPS mit Root-Zugang; eine Managed-Plattform ist nicht nötig. Jeder Anbieter mit Docker und öffentlicher IP genügt. Größenordnung aus unserem Betrieb: Für die App reicht eine kleine Instanz, aber **gib der Festplatte mehr Luft, als nötig erscheint**, denn der oben beschriebene Layer-Export-Ausfall ist ein Speicher-, kein CPU-Problem. Plan 20 GB Puffer über deine Images hinaus ein.

Unsere laufen bei Hostinger. **Empfehlungslink — [hostinger.com](https://www.hostinger.com/tr?REFERRALCODE=KAWDURSUNLTO)** — seine Nutzung bringt dem Autor eine Provision und dir einen Rabatt. Es ist keine Voraussetzung: Coolify läuft bei jedem Anbieter mit Docker und Root-Zugang, und nichts in diesem Leitfaden hängt vom Hoster ab.

---

## Wie das mit der Prüfung zusammenhängt

Mehrere Store-Ablehnungen aus dem Katalog waren Infrastrukturprobleme im Kostüm einer Guideline-Nummer:

| Sah aus wie | War tatsächlich |
|---|---|
| Guideline 1.2, keine Möglichkeit, Inhalte zu melden | Eine veröffentlichte Kontaktadresse ohne MX-Eintrag |
| Play-Einreichung abgelehnt | Die angegebene Datenschutz-URL lieferte 404 |
| 2.1 App Completeness, „App stürzt beim Start ab" | Die `.env` erreichte den Build nie |
| 2.1, „wir konnten die Funktion nicht erreichen" | Ein in der Produktion abgeschaltetes Feature-Flag |

Bevor du der Prüfung die Schuld gibst: prüfe, ob das, was sie nicht erreichen konnte, von außerhalb deiner Maschine wirklich erreichbar ist.
