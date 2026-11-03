# Job Tracker

Single-File-PWA aus der mvs2-App-Familie: alle Bewerbungen an einem Ort. Wo und wann beworben,
wie der Stand ist, welche Fristen laufen, was in der Anzeige stand und mit welchem Benutzernamen
man ins Bewerberportal kommt. Gleiche Designsprache und gleicher Kontozugang wie MigraLog und TravelBase.

Live: https://mvs2-bot.github.io/JobTracker/

## Dateien

| Datei | Inhalt |
| --- | --- |
| `index.html` | die komplette App: HTML, eigenes CSS, Alpine.js und supabase-js über CDN |
| `manifest.json`, `sw.js` | installierbar (eigene `id: jobtracker-app`), App-Hülle offline verfügbar |
| `icon.svg`, `icon.png`, `apple-touch-icon.png`, `icons/` | App-Icon im Langschatten-Stil (Aktentasche auf Petrol), maskable und Favicon |
| `shared/ios-nav.js` | Kopie aus `mvs2-bot.github.io` (Zurück/Neu laden in der iOS-Web-App) |
| Datenbank | Migration `…_jobtracker_core` im Repo `supabase-core` |

## Bereiche

- **Übersicht:** Kennzahlen (offen, Gespräche, Absagen, Antwortquote, Ø Tage bis zur Antwort),
  die nächsten drei Termine und die zuletzt geänderten Bewerbungen.
- **Bewerbungen:** Suche, Filter nach Status, Archiv ein/aus. Auf dem Desktop (ab 1024 px)
  Liste links und Detail rechts oder Spaltenansicht nach Status.
- **Demnächst:** alle offenen Fristen, gruppiert nach überfällig, heute, nächste 7 Tage, später.

Eine Bewerbung hat Firma, Stelle, Status, Eckdaten (Ort, Arbeitsmodell, Datum, Quelle, Gehaltswunsch,
Link), Fristen und To-dos, den Status-Verlauf mit änderbarem Datum, Ansprechperson, Bewerberportal,
eine Kopie der Stellenanzeige und Notizen. Gespeichert wird automatisch beim Tippen.

## Kennzahlen

- **Offen:** nicht archiviert, Status Beworben bis Angebot.
- **Antwortquote:** Bewerbungen mit einer Antwort (Gespräch, Angebot, Zusage oder Absage) geteilt durch
  alle beworbenen. Die Eingangsbestätigung zählt nicht als Antwort.
- **Ø Tage bis zur Antwort:** vom Bewerbungsdatum bis zum ersten Antwort-Eintrag im Verlauf.

## Daten und Offline

- Tabellen `jobtracker_applications`, `jobtracker_events` (Status-Verlauf, schreibt ein Trigger),
  `jobtracker_tasks`. IDs erzeugt die App (`crypto.randomUUID()`).
- Ohne Netz landen Änderungen in `localStorage['jobtracker-queue-<user>']` (Liste in Sende-Reihenfolge)
  und werden nachgeschickt. Der letzte Stand liegt ohne Anzeigentexte in `jobtracker-cache-<user>`,
  damit die App auch offline startet.
- Supabase-Anfragen cacht der Service Worker nie.

## Rollen und Datenschutz

- Zugang mit mindestens Rolle **Mitglied** für die App `jobtracker` (Admin-Einstellungen im Dashboard).
- Bewerbungen sind **strikt privat** (RLS): auch Admins sehen keine fremden Bewerbungen.
- **Keine Passwörter:** Für Bewerberportale speichert die App nur Link und Benutzername.
  Das Passwort bleibt im Passwort-Manager.
- Name, Profilbild und Hell/Dunkel kommen aus dem gemeinsamen Konto (`overall_profiles`, `overall-theme`).

## Design

Systemschrift, weiße Karten (18 px Radius) auf `#f2f2f7`, dunkel `#1c1c1e` auf Schwarz.
Akzent Petrol `#0b6e79` (dunkel `#5cc8d4`). Status-Farben: grau gemerkt/zurückgezogen, Petrol laufend,
Grün Angebot/Zusage, Rot Absage. Fristen: Rot überfällig, Orange heute.

## Ändern

Bei jeder Änderung an der App die Cache-Version in `sw.js` (`V=P+"v…"`) hochzählen.
