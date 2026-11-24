# CLAUDE.md – JobTracker

Single-File-PWA der mvs2-App-Familie: Bewerbungen, Status-Verlauf, Fristen, Stellenanzeigen. Live unter `/JobTracker/`.
Kein Build, kein npm: reines HTML/CSS/JS, Alpine.js + supabase-js per CDN. Sprache in UI, Kommentaren und Commits: Deutsch.
Produktentscheidungen in `PRODUCT.md`, Nutzersicht in `README.md`.

## Arbeitsweise (Token-Effizienz)
- `index.html` (~1440 Zeilen) nie komplett lesen. Per `grep -n` den Abschnitt anspringen (Karte unten).
- Vor dem PR selbst prüfen: JS-Syntax, Alpine-Bindings ↔ Methoden/Felder, IDs, Tokens statt harter Farben,
  Spalten ↔ Schema in `supabase-core`, Hell + Dunkel, Handy (< 768 px), Tablet hoch (768–899 px) + Desktop (≥ 900 px).
- **Bei jeder Änderung `V=P+"vNN"` in `sw.js` hochzählen.** Neue Datei → in `CORE` von `sw.js` aufnehmen.
- Bei Änderung an Architektur, Struktur oder Schema diese Datei im selben PR anpassen (max. 150 Zeilen).
- Keine Secrets. Im Code ist nur der öffentliche Publishable Key erlaubt, niemals `service_role`/secret Key.

## Dateien
| Datei | Inhalt |
| --- | --- |
| `index.html` | komplette App: Markup, CSS, Konstanten, Alpine-Komponente `jobTracker()` |
| `sw.js` | Service Worker: eigene Dateien network-first (4 s), CDN cache-first, Supabase nie cachen |
| `manifest.json` | PWA, `id: jobtracker-app`, `start_url`/`scope` `./` |
| `icon.svg`, `icon.png`, `apple-touch-icon.png`, `icons/` | App-Icon (Aktentasche, Langschatten auf Petrol), maskable, Favicon |
| `shared/ios-nav.js` | **Kopie** aus `mvs2-bot.github.io/shared/` (Zurück/Neu laden in iOS-Web-App). Nur im Original ändern. |
| `README.md`, `PRODUCT.md` | Funktionsbeschreibung, Kennzahl-Definitionen, Entscheidungen, Ideen für später |

## Karte von `index.html`
| Zeilen ca. | Inhalt |
| --- | --- |
| 19–26 | Theme vor dem Rendern (`localStorage['overall-theme']` → Klasse `.dark` auf `<html>`), `grow()` für Textareas |
| 28–395 | CSS: Tokens `:root`/`.dark`, Komponenten, `@media (min-width:900px)` Desktop (auch Tablet quer, JS `mq` gleich), `(max-width:767.98px)` Handy |
| 404–468 | Konstanten: Supabase-Projekt, `APP_ID`, `IC` (SVG-Symbole), `STATUSES`, `FILTERS`, `COLUMNS`, `MODES`, `KINDS`, `SOURCES`, `*_FIELDS` |
| 473–905 | Markup: Header, Gates (`loading`/`auth`/`noaccess`), Tabs `home`/`apps`/`soon`/`new`/`profile`, Tabbar |
| 909–1435 | Logik: Datum-Helfer, `jobTracker()` mit Abschnitten Start, Anmeldung, Laden/Cache, Speichern/Warteschlange, Fristen, Navigation, Listen/Kennzahlen, Sync-Anzeige, Profil, CSV |

Abschnitte finden: `grep -n "// [A-Z]" index.html`, Methoden: `grep -n "^      \(async \)\?[a-zA-Z]*(.*) {" index.html`.

## Zustand und Ablauf
- `view`: `loading | auth | noaccess | app`; `tab`: `home | apps | soon | new | profile` (letzter Tab in `sessionStorage['jobtracker-tab']`).
- Start: Session (Timeout 8 s) → `loadMe()` (Profil, Rolle) → `checkAccess()` (RPC `jobtracker_is_member`) → `loadAll()`.
  Start-Wächter `bootMsg()` schreibt direkt ins DOM, falls Alpine hängt. Netz immer über `withTimeout()`.
- Daten im Speicher: `apps`, `events`, `tasks`. IDs erzeugt die App (`crypto.randomUUID()`).
- Schreiben nur über `run(op)` (`{ t: <tabelle>, op: 'upsert'|'delete', row }`): erst `applyLocal`, dann `send`;
  offline → `enqueue` in `localStorage['jobtracker-queue-<uid>']`, `flushQueue()` schickt nach.
- Felder vor dem Senden mit `pick(row, APP_FIELDS|TASK_FIELDS|EVENT_FIELDS)` filtern (`''` → `null`).
  Neue Spalte = Migration in `supabase-core` **und** Eintrag in der passenden `*_FIELDS`-Liste.
- Tippen speichert automatisch (`queueSave` mit Timer, `saveAllNow` beim Verlassen).
- Offline-Start aus `jobtracker-cache-<uid>` (ohne Anzeigentexte), Profil aus `jobtracker-me-<uid>`, Board-Ansicht `jobtracker-board`.
- Status-Logik: `CLOSED`, `RUNNING`, `ANSWERS` (Eingangsbestätigung zählt nicht als Antwort). Definitionen der Kennzahlen in `README.md`.

## Supabase-Anbindung
- Gemeinsames Core-Projekt (`SUPABASE_URL`/`SUPABASE_ANON_KEY` in `index.html`), supabase-js `2.45.4`, Alpine `3.14.1` (jsDelivr).
  CDN-Versionen auch in `EXTRA` von `sw.js` pflegen.
- **Schema liegt ausschließlich im Repo `supabase-core`** (Migration `20261103090000_jobtracker_core.sql`, Test `jobtracker_core_test.sql`).
  Schemaänderung = dort neue Migration (Präfix `jobtracker_`), eigener abgestimmter PR.
- Tabellen: `jobtracker_applications`, `jobtracker_events` (Status-Verlauf, schreibt Trigger `jobtracker_log_status` bei Insert/Statuswechsel),
  `jobtracker_tasks`. `created_at`/`updated_at` setzt die DB. Check-Constraints (Status, `work_mode`, `kind`, Längen) beachten.
- RPCs: `jobtracker_is_member`, `overall_app_role(p_app)`; außerdem `overall_profiles` (Name, Avatar) und Storage-Bucket `avatars` (signierte URL).
- Zugriff: App `jobtracker` ab Rolle `member`. RLS **strikt privat** pro `user_id`, auch Admins sehen keine fremden Daten.
- Keine Passwörter speichern: Bewerberportal nur Link + Benutzername (Entscheidung in `PRODUCT.md`).

## Schnittstellen zum Dashboard (`mvs2-bot.github.io`)
- `shared/projects.js`: Kachel (`app: 'jobtracker'`, `kpi: true`, Farbe `#0b6e79`).
- `shared/kpi.js` + RPC `overall_tile_kpis` (Migration `…_overall_tile_kpis.sql`): Chip „N offen“ (`open`).
- `konto.html`: Karte `jobtrackerCard` (Anzahl Bewerbungen, liest `jobtracker-queue-<uid>`). Links aus der App:
  `ACCOUNT_URL` (`/konto.html?from=jobtracker#app-jobtracker`), `ADMIN_URL` (`/admin.html?from=jobtracker`).
- `admin.html`: Rechte-Beschreibung je Rolle; `usage.html`: Label in der Nutzung.
- Ändert sich Definition „offen“, Queue-Key oder Tabellenname → Dashboard und `overall_tile_kpis` mit anpassen.

## Design-Tokens (`:root` in `index.html`, dunkel unter `.dark`)
- Flächen: `--bg #f2f2f7`, `--card #fff`, `--well`, `--hover`, `--line`, `--sep`.
- Text: `--ink #1c1c1e`, `--text #3a3a3c`, `--mute #6c6c70`.
- Akzent Petrol: `--p1 #0b6e79` (dunkel `#5cc8d4`), `--p1-soft`, `--on-p1`.
- Status: `--ok`, `--ok-soft`, `--warn`, `--warn-soft`, `--danger`, `--danger-soft`.
- `--radius 18px`, `--radius-s 12px`, `--shadow`, `--pop`, `--font` (Systemschrift/SF), `--add-tint`.
- Aliasse für `ios-nav.js`: `--border`, `--accent`, `--card-hover`, `--muted`.
- Status-Chips: `''` grau (Gemerkt/Zurückgezogen), `s-run`/`s-talk` Petrol, `s-ok` Grün, `s-no` Rot. Fristen: Rot überfällig, Orange heute.
- Apple-Designsprache wie Dashboard/MigraLog/TravelBase: weiße Karten auf Grau, iOS-Segmente, Linien-Symbole aus `IC`, keine Emojis.
- Neue Farben als Token in `:root` **und** `.dark`, nicht hart im Markup. Dark Mode über Klasse `.dark` (nicht `data-theme`).

## Konventionen
- Kalenderdaten immer lokal (`toISO`, `fromISO`, `addDays`, `dayDiff`); `toISOString()` nur für Zeitstempel (`updated_at`, `archived_at`).
- `localStorage`-Zugriffe in `try/catch`. Keys mit Präfix `jobtracker-`, Theme bleibt `overall-theme` (app-übergreifend).
- Neue Symbole in `IC` mit `svg(...)` (24er-ViewBox, Linien-Stil). Neuer Status → `STATUSES`, ggf. `FILTERS`/`COLUMNS`/`CLOSED`/`RUNNING`/`ANSWERS` **und** Check-Constraints in der DB.
- Geteilte Dateien: Original ändern, dann Kopien in allen Repos aktualisieren (`ios-nav.js` aus `mvs2-bot.github.io`).
