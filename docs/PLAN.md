# MovieTracker – Umsetzungsplan

> Stand: 2026-10-27 · Status: **Phase 0** (Umgebung)
> Grundlage: das ursprüngliche Projekt-Briefing (nur lokal, nicht Teil des Repos; Verweise wie „Briefing §6“ beziehen sich darauf). Dieser Plan hält fest, was in der Planungsphase zusätzlich entschieden wurde und in welcher Reihenfolge gebaut wird. Wo er vom Briefing abweicht, steht die Begründung dabei.
> Sprache der Projektdokumente: Deutsch, Bezeichner englisch im Code-Format. Code, Kommentare, Commits: Englisch.

---

## 1. Entscheidungen aus der Planungsphase

| Thema | Entscheidung |
|---|---|
| Technologie | wie Briefing §2 (Flutter, Dart, drift + SQLCipher, Argon2id, gen-l10n) |
| Datenbank | **SQLite + drift + SQLCipher bestätigt** nach Vergleich mit Document Stores (Isar, Hive, ObjectBox, Sembast, Realm: schwache/fehlende Verschlüsselung, unsichere Wartung) und einer verschlüsselten JSON-Datei. Die JSON-Datei ist **Rückfallebene**, falls sich SQLCipher unter Linux nicht sauber mitliefern lässt (Spike) |
| Sprache & Theme | **Abweichung von Briefing §2/§8:** Erstes Release nur **Deutsch** und nur **Dark Mode**. Englisch und Light Mode kommen vielleicht in einem späteren Release. Die Architektur ist von Anfang an darauf vorbereitet (s. §3.5) |
| Einrichtungsassistent | **Abweichung von Briefing §6:** Beim ersten Start wird „Daten mit Passwort schützen?“ abgefragt, **ohne Vorauswahl**. Der Nutzer muss aktiv Ja oder Nein wählen; der Text erklärt kurz Nutzen und Risiko |
| Design | **offen**, wird in Phase 2 mit mehreren Konzepten erarbeitet (s. §3). Bisher bevorzugt, aber nicht festgelegt: Liste + Detail |
| Filme / Serien | Trennung im UI ist ein wichtiges Feature, eine gemeinsame Ansicht für alles ist ebenfalls gewünscht. Trennkonzepte werden im Design erarbeitet (s. §3.2) |
| Bilder/Poster | nicht geplant |
| Statistiken | nicht im MVP, aber nicht ausgeschlossen. Sinnvolle Auswertungen später prüfen (s. §4) |
| Seh-Historie (jeder Fortschritt) | nicht gewünscht |
| Erscheinungstermine | Optionales voraussichtliches Erscheinungsdatum für noch nicht erschienene Filme/Serien und für die nächste Staffel bei `awaiting_episodes`. Ansichten „Demnächst“ / „Neu erschienen“ und Hinweise. Wird in Phase 2 mit Mockups konkret spezifiziert (s. §3.4); ob MVP oder später, wird dort entschieden |
| Aufnahmeschutz (Windows) | Einstellung, Standard: aus |
| Bewertung | NULL = keine Bewertung, sonst 0–10 in 0,1er-Schritten. Gespeichert als Ganzzahl in Zehnteln (`rating_tenths` 0–100), damit spätere Umstellung (0,5er / Sterne) nur die Anzeige betrifft |
| Status | `watchlist`, `watching`, `paused`, `awaiting_episodes` (nur Serien: alles Erschienene gesehen, wartet auf neue Folgen), `dropped`, `watched` |
| Rewatch | Durchgänge mit optionalem Datum werden erfasst, Anzahl = Zahl der Durchgänge (s. §4) |
| App-Name | „MovieTracker“ bleibt (auch Exe-Name). Zu schützen sind die **Inhalte der DB**, insbesondere Links |
| Dateinamen | **Abweichung von Briefing §3.2:** Die Datenbank heißt `movietracker.db` statt `vault.db`. Sie liegt ohnehin neben der Exe, ein neutraler Name verbirgt nichts und „vault“ wirkt eher auffällig. **Exporte** bleiben neutral benannt, weil sie ohne Ordner-Kontext weitergegeben werden |
| Linux | **WSL (Ubuntu) und GitHub-Repo mit Actions**, beides ab Phase 0 |
| Repo & Weitergabe | Repo wird **am Ende von Phase 0 öffentlich** (kostenlose Actions-Minuten, Branch-Schutz, Secret Scanning). Vorher: Lizenz, zweisprachige README, Dokumente auf Veröffentlichung geprüft. Issues aus, solange keine Rückmeldungen von außen gewünscht sind. App wird persönlich weitergegeben. **GitHub Releases** ab dem ersten Release, mit SHA-256-Prüfsummen |
| Branches | `main` ist immer grün; gearbeitet wird auf kurzlebigen Branches je Schritt (z. B. `phase-3/data-dir`), zurück per Pull Request mit grüner CI und **Rebase and merge** (kleine Commits bleiben einzeln erhalten). Kein `dev`-Branch: Releases sind Tags auf `main` (`v1.0.0`). Der Spike (Phase 1) liegt auf `spike/phase-1` und wird nie gemergt |
| Privatsphäre des Entwicklers | **Der volle Name des Nutzers darf nirgends landen**: nicht in Code, Doku, Commits (Autor = GitHub-noreply-Adresse), Testdaten, Pfaden, Build-Artefakten. Persönliches nur in `CLAUDE.local.md` und `.claude/settings.local.json` (beide per `.gitignore` ausgeschlossen). Das ursprüngliche Briefing bleibt lokal; dauerhaft Gültiges wird in Phase 2 ohne persönliche Angaben in `SPEC.md` / `CLAUDE.md` übernommen |
| Doku | Deutsch, im Repo: `SPEC.md`, `CLAUDE.md`, `docs/`. README zweisprachig (Deutsch/Englisch) |
| Lizenz | **PolyForm Strict 1.0.0**. Kann später geöffnet werden (z. B. GPLv3), solange der Nutzer alleiniger Rechteinhaber ist → keine fremden Beiträge ohne Zustimmungsvereinbarung |
| Code-Signierung | keine. README erklärt SmartScreen-Hinweis und „Zulassen“; Releases mit SHA-256-Prüfsummen |
| Updates | manuell: Ordner `app-windows/` bzw. `app-linux/` ersetzen, `data/` bleibt. Ältere App-Versionen verweigern mit klarer Meldung das Öffnen einer DB mit neuerem Schema. Update-Prüfung auf Knopfdruck evtl. später |
| Flutter SDK | stable, festgeschrieben auf eine Version (Phase 0: 3.47.8; in der CI per `FLUTTER_VERSION`). Windows: `C:\dev\tools\flutter`, Paket-Cache `C:\dev\tools\pub-cache` (`PUB_CACHE`, damit der Benutzerpfad nicht in generierte Dateien gerät). WSL: `~/tools/flutter`. Analytics von Flutter und Dart überall aus |
| Automatische Sicherung | Vor jeder Schema-Migration und vor dem Ein-/Ausschalten der Verschlüsselung legt die App eine Kopie in `data/backups/` an; ältere Kopien werden aufgeräumt. Klartext-Kopien werden beim Aktivieren der Verschlüsselung gelöscht |
| Datenübernahme | Einmalige Übernahme aus einer früheren, privaten Version (s. §2.5) |

---

## 2. Abweichungen und Ergänzungen zum Briefing

> Alle Punkte 2.1–2.8 vom Nutzer freigegeben (2026-10-27).

### 2.1 Durchgehendes Grundgerüst statt Schicht für Schicht
Briefing §11.3 baut Schicht für Schicht, Packaging kommt zuletzt. Neu: Zuerst entsteht ein **durchgehendes Minimal-Gerüst** (DB → Liste → Anlegen → **gepacktes zip/tar.gz** → Portabilitätstest auf beiden Systemen). Danach wachsen Features einzeln durch alle Schichten.
**Grund:** Datenordner-Ermittlung (`<exe>/../data`) und Pfad-Umleitung lassen sich nur im gepackten Layout echt prüfen.

### 2.2 Eine zentrale Stelle für alle Schreibzugriffe
Die UI schreibt nie direkt in die DB. Jede Änderung läuft durch eine Schicht mit Kommandos (`updateProgress`, `setStatus`, `deleteItem`, …). Diese Schicht setzt `updated_at`, macht Soft-Deletes (auch für `media_tags`) und ist später der Anknüpfungspunkt für den Sync.
**Grund:** Sync-Regeln aus Briefing §5 werden an einer Stelle erzwungen statt in jedem Screen.

### 2.3 Kern als reines Dart-Paket
Monorepo: `packages/core` (Domäne, drift-Schema, Repository, Import/Export, Krypto, ohne Flutter) und `app/` (Flutter-UI, Runner, Plattformcode), dazu `tools/`.
**Grund:** schnelle Tests mit `dart test`, wiederverwendbar für Android und CLI-Werkzeuge, Trennung UI/Domäne technisch erzwungen.

### 2.4 Schlüsselarchitektur: Datenschlüssel einhüllen
Briefing §6 leitet den DB-Key direkt aus dem Passwort ab und will gleichzeitig einen Wiederherstellungsschlüssel. Das geht so nicht zusammen. Neu:
- Die DB wird mit einem **zufälligen Datenschlüssel** verschlüsselt.
- Dieser liegt in `bootstrap.json` zweimal **eingehüllt** (verschlüsselt): einmal mit dem Argon2id-Schlüssel aus dem Passwort, einmal mit dem Wiederherstellungsschlüssel.
- Passwort ändern = nur neu einhüllen (ohne `PRAGMA rekey` der ganzen DB).
- `bootstrap.json` enthält damit: `encrypted`, KDF-Parameter, Salt, eingehüllte Schlüssel. Die Schema-Version entfällt dort (steht in der DB als `user_version`).

### 2.5 Datenübernahme als lokaler Konverter statt App-Feature
Daten aus einer früheren, privaten Version werden einmalig über einen lokalen Konverter ins **JSON-Exportformat** gebracht und über den normalen Backup-Import übernommen. Der Konverter ist nicht Teil dieses Repos.
**Grund:** kein Code für ein Altformat in der App; der Backup-Import wird nebenbei mit echten Daten getestet.

### 2.6 Verschlüsselung: Kern früh, UI später
Öffnen mit oder ohne Schlüssel, Einhüllen und `sqlcipher_export` entstehen in Phase 3 in der Datenschicht. Assistent, Auto-Sperre und Einstellungen dazu kommen in Phase 6, vor dem ersten Release.

### 2.7 Rollback-Journal statt WAL, Instanzsperre
`journal_mode = DELETE`: im Ruhezustand nur eine Datei, kein Shared Memory, robust auf exFAT/FAT in VeraCrypt-Containern. Eine Sperrdatei (`data/.lock`) verhindert, dass zwei Instanzen gleichzeitig auf die DB zugreifen.

### 2.8 Windows-Portabilitätstest automatisiert
Process Monitor per Kommandozeile (Mitschnitt mit Prozessfilter → CSV → Auswertung) statt Snapshot-Vergleich + manueller Checkliste. Braucht Admin-Rechte.

---

## 3. Design-Exploration (Phase 2)

Das Design ist **nicht festgelegt**. In Phase 2 entstehen klickbare Mockups mit unterschiedlichen Konzepten. Erst danach wird entschieden.

### 3.1 Prüffälle für jedes Konzept
Jedes Konzept muss zeigen, wie diese vier Abläufe funktionieren und wie viele Schritte sie brauchen:

| Ablauf | Anforderung |
|---|---|
| Link holen | Eintrag finden und Link kopieren (geschützt, s. [SECURITY-CONCEPT.md](SECURITY-CONCEPT.md)), möglichst ohne Maus |
| Fortschritt nachtragen | Folge / Staffel / Position schnell ändern, Rückgängig möglich |
| Merkliste pflegen | Neuen Titel mit minimalen Angaben erfassen |
| Stöbern / Auswählen | Überblick über Merkliste / Gesehenes, filtern, sortieren |

Außerdem: adaptiv (breites Fenster ↔ schmales Fenster / Android), Liste ↔ Kacheln, Dark Mode (Farben mit Bedeutung zusätzlich auf hellem Grund prüfen, s. §3.5).

### 3.2 Trennung Filme / Serien
Anforderung: Getrennte Ansichten sind ein wichtiges Feature, eine gemeinsame Ansicht für alles soll es auch geben. Zu erarbeitende Konzepte:

| Konzept | Beschreibung |
|---|---|
| Tabs | „Alle · Filme · Serien“ als Reiter über der Liste |
| Segment-Schalter | Kompakter Umschalter mit drei Stellungen, z. B. in der Kopfzeile |
| Seitenleiste | Filme und Serien als eigene Bereiche in der Navigation, „Alle“ als dritter Bereich |
| Filter-Chips | Typ ist ein Filter unter mehreren (neben Status, Favorit, Tags), kombinierbar |
| Gruppierte Liste | Eine Liste mit Abschnitten „Serien“ / „Filme“, einklappbar |
| Zwei Spalten | Filme und Serien nebeneinander auf breiten Fenstern |
| Getrennte Arbeitsbereiche | Zwei eigenständige Bereiche mit eigener Sortierung und eigenem Filter, „Alle“ nur über die Suche |
| Typ nur als Kennzeichen | Eine gemeinsame Liste, Typ als Symbol/Farbe; Trennung nur per Tastenkürzel oder Suche (`f:` / `s:`) |

Dazu für jedes Konzept: Merkt sich die App die letzte Auswahl? Gilt die Trennung auch für die Suche?

### 3.3 Grundkonzepte für das Hauptfenster
Schwerpunkt: Varianten von **Liste + Detail**. Zusätzlich als „wie es ganz anders aussehen könnte“:
- **Suche zuerst** (Launcher-Stil, Tastatur zentriert)
- **Board nach Status** (Spalten, Karten verschieben)
- weitere Ideen, die sich beim Entwerfen ergeben

### 3.4 Erscheinungstermine
Zu spezifizieren mit Mockups:
- **Erfassung:** Mindestens **Monat + Jahr**, optional der genaue Tag. Wie gibt man das schnell ein, wie wird es angezeigt?
- **Noch nicht erschienene Titel:** Werden sie in der Merkliste mitgeführt (Kennzeichen „angekündigt“) oder getrennt angezeigt? Abgeleitet aus dem Datum, kein eigener Status.
- **`awaiting_episodes`:** Datum der nächsten Staffel/Folge, optional mit Angabe, was erscheint („Staffel 3“).
- **Ansichten (gewünscht):** „Demnächst“ (nach Datum sortiert) und „Neu erschienen“ (Datum erreicht, noch nicht reagiert). Was zählt als „reagiert“: Status geändert, Hinweis weggeklickt?
- **Hinweise (entschieden):** **nur innerhalb der App**, keine System-Benachrichtigungen. Gestaltung offen (z. B. Banner oder Zähler „2 neu erschienen“).
- **Später:** Daten optional über einen Metadatendienst ergänzen (nur auf ausdrückliche Aktion, Briefing §7).

### 3.5 Sprache und Theme: jetzt Deutsch + Dark, vorbereitet für Englisch + Light
Damit Englisch und Light Mode später ohne Umbau dazukommen:
- **Texte:** Von Anfang an ausschließlich über ARB-Dateien (`gen-l10n`), zunächst nur `app_de.arb`. Keine Texte im Code (Lint + Test).
- **Formate:** Datum, Zahlen, Bewertungen über `intl` mit der aktiven Sprache, nicht hartcodiert („12.03.2027“, „8,5“).
- **Pseudo-Sprache für Tests:** Eine Test-Sprache, die jeden Text verfremdet und verlängert (z. B. „[Ŝţåţûš ~~~]“), macht vergessene hartcodierte Texte und zu knappe Layouts sichtbar. Wichtig, weil englische und deutsche Texte unterschiedlich lang sind.
- **Farben:** Nur über semantische Design-Tokens (z. B. `surface`, `textPrimary`, `accent`) in einer `ThemeExtension`, nie direkt im Widget. Light Mode ist dann ein zweiter Satz Token-Werte.
- **Design:** Mockups in Phase 2 dark-first. Wo Farben Bedeutung tragen (Status, Hinweise), wird kurz geprüft, dass sie auch auf hellem Hintergrund funktionieren würden.
- **Einstellungen:** Sprache und Theme erscheinen erst in den Einstellungen, wenn es eine Auswahl gibt. Speicher dafür ist in `settings` vorgesehen.

---

## 4. Datenmodell – Änderungen gegenüber Briefing §5

- `media_items.status`: `watchlist | watching | paused | awaiting_episodes | dropped | watched`. `awaiting_episodes` nur bei `type = series` (Validierung im Kern).
- **Neu `media_items.status_changed_at`**: Zeitpunkt des letzten Statuswechsels. Kostet nichts und ermöglicht Anzeigen wie „seit 8 Monaten pausiert“ oder „seit 2 Jahren auf der Merkliste“.
- `media_items.rating` → `rating_tenths INTEGER NULL` (0–100).
- **Neu: Erscheinungstermin** (Entwurf, wird in Phase 2 nach den Mockups festgelegt): Datum mit Genauigkeit (Monat oder Tag; mindestens Monat + Jahr), optionale Bezeichnung („Staffel 3“), Zeitpunkt, zu dem der Nutzer auf „neu erschienen“ reagiert hat.
- **Neu `viewings`**: `id` (UUID), `media_id`, `completed_on` (Datum, NULL = unbekannt), `created_at`, `updated_at`, `deleted_at`.
  - Anzahl der Durchgänge = Zahl der nicht gelöschten Zeilen.
  - Wechsel auf `watched` → App bietet an, einen Durchgang zu speichern (Datum vorbelegt mit heute).
  - Rewatch = Status wieder auf `watching`, nach Abschluss neuer Durchgang.
  - Frühere Durchgänge mit oder ohne Datum nachtragbar.
- `media_tags`: zusätzlich `created_at`, `updated_at`, `deleted_at` (Soft-Delete für Sync).
- `settings`: zusätzlich `device_id` (UUID, beim ersten Start erzeugt), Vorbereitung für Sync.

**Statistiken (später, nicht im MVP):** Mit diesem Modell sind ohne weitere Datenerfassung möglich: Anzahl je Status und Typ, Durchgänge pro Jahr, häufigste Rewatches, Bewertungsverteilung, Tags nach Häufigkeit, „Aufräum-Hinweise“ (lange pausiert, lange auf der Merkliste). Statistiken, die jeden Fortschritt bräuchten (z. B. Folgen pro Woche), sind bewusst nicht möglich.

---

## 5. Datenschutz und Sicherheit

Vollständiges Sicherheitskonzept mit Schutzmaßnahmen, Technik und Restrisiken: [SECURITY-CONCEPT.md](SECURITY-CONCEPT.md).

---

## 6. Phasen

Nach jeder Phase und innerhalb der Phasen 3–7 nach jedem größeren Schritt: alle Tests + Portabilitätstests ausführen, Ergebnis berichten, auf Freigabe zum Commit warten.

### Phase 0 – Umgebung
- Flutter SDK (stable), Visual Studio 2022 mit „Desktopentwicklung mit C++“
- WSL mit Ubuntu (22.04, wegen glibc-Kompatibilität), darin Flutter, `bwrap`, `strace`
- Process Monitor (Sysinternals)
- `git init`, `.gitignore` ergänzen (u. a. `build/`, `.dart_tool/`, Test-Artefakte; persönliche Dateien sind bereits ausgeschlossen), `.gitattributes` (LF), GitHub-Repo, CI-Grundgerüst (Linux- + Windows-Job)
- Git für dieses Repo: `user.name` = GitHub-Benutzername, `user.email` = GitHub-noreply-Adresse; auf GitHub „Keep my email address private“ und „Block command line pushes that expose my email“ aktivieren
- **Lokale Hooks** `pre-commit` + `commit-msg` (liegen in `.git/hooks`, werden nicht versioniert, Muster in `.git/hooks/privacy-patterns`): brechen den Commit ab, wenn der volle Name oder die echte E-Mail-Adresse in Dateinamen, geänderten Inhalten (auch binär), Autor/Committer oder der Commit-Message vorkommt
- Veröffentlichung vorbereiten: `LICENSE.md`, zweisprachige `README.md`, Dokumente geprüft. Dann schaltet der Nutzer das Repo selbst auf öffentlich; danach Branch-Schutz für `main`, nur „Rebase and merge“, Secret Scanning, Issues aus
- Release-Builds entstehen nur in der CI, nicht lokal (Flutter/Dart können Build-Pfade in Binaries einbetten, z. B. in Stacktraces oder Debug-Infos). Prüfung der Artefakte s. Phase 3

**Abnahme:** `flutter doctor` grün für Windows (Host) und Linux (WSL); CI läuft auf einem leeren Projekt durch.

### Phase 1 – Spike (Wegwerf-Code)
Klärt drei Risiken mit einer Mini-App (Fenster auf, verschlüsselte DB anlegen, Datensatz schreiben, beenden):
1. drift + SQLCipher **mitgeliefert** auf Windows und Linux (keine Systembibliothek nötig)?
2. Argon2id: welches Paket, wie lange dauert das Ableiten bei sinnvollen Parametern?
3. Greift die Pfad-Umleitung im nativen Runner früh genug? Was schreiben Grafiktreiber trotzdem?

Außerdem: Machbarkeit des geschützten Kopierens und des Aufnahmeschutzes für das Fenster (SECURITY-CONCEPT.md §4.3, §4.4).

**Abnahme:** kurzer Bericht mit Messwerten → feste Paketliste zur Freigabe.

### Phase 2 – Spezifikation & Design
- `SPEC.md`: Datenmodell, Screens, Abläufe, Settings, Fehlerverhalten
- `CLAUDE.md`: harte Regeln (Briefing §3), Befehle, Projektstruktur
- Design-Mockups (klickbar) nach §3: Grundkonzepte, Trennkonzepte Filme/Serien, visuelle Varianten, jeweils gegen die Prüffälle aus §3.1

**Abnahme:** SPEC und Designrichtung freigegeben.

### Phase 3 – Grundgerüst
Monorepo (`core`, `app`, `tools`), Lints, l10n, Datenordner, Runner-Umleitung, Instanzsperre, Schreib-Kommandos, DB öffnen mit/ohne Schlüssel + Einhüllen, Migrationen, Selbsttest-Modus, **Packaging-Skripte**, Portabilitätstests (lokal + CI), Prüfung der gepackten Artefakte auf eingebettete lokale Pfade und persönliche Daten.

**Abnahme:** gepacktes zip/tar.gz kann einen Eintrag anlegen und anzeigen; beide Portabilitätstests grün; Artefakte frei von lokalen Pfaden.

### Phase 4 – Kernfunktionen
Hauptfenster nach freigegebenem Design, Liste/Kacheln, Suche/Filter/Sortierung, Detail/Bearbeiten mit Validierung, Schnellaktionen, Durchgänge, Schnellerfassung, Löschen + Rückgängig, geschütztes Kopieren.

**Abnahme:** Widget-Tests der vier Abläufe; Selbsttest deckt alle Kommandos ab.

### Phase 5 – Daten
JSON-Export/-Import, Einstellungen, Datenübernahme aus der früheren Version (§2.5).

**Abnahme:** Roundtrip-Test grün.

### Phase 6 – Verschlüsselung & Logs
Assistent, Ein/Aus, Passwort ändern, Wiederherstellungsschlüssel, Auto-Sperre als Einstellung (Inaktivität, Minimieren, Ruhezustand), Argon2id + Passwortempfehlung, verschlüsselte Exporte, Aufnahmeschutz (Windows), Logger.

**Abnahme:** Klartext-, Log- und Roundtrip-Tests grün.

### Phase 7 – Feinschliff & v1.0
Animationen, Befehlspalette, Tastaturkürzel, Hilfe-Seite mit Grenzen, Lizenzen.

**Abnahme:** Release-zip (Windows) und Release-tar.gz (Linux) aus der CI.

### Später
Android → Sync → ggf. Metadaten. Die Architektur (§2.2, §2.3) ist darauf vorbereitet.

---

## 7. Risiken

| Risiko | Auswirkung | Gegenmaßnahme |
|---|---|---|
| SQLCipher unter Linux nur als Systembibliothek | App startet auf fremden Rechnern nicht | Spike (Phase 1), notfalls selbst bauen und mitliefern; Rückfallebene: verschlüsselte JSON-Datei |
| glibc-Version des Linux-Builds | läuft nicht auf älteren Distros | Release-Build auf Ubuntu 22.04 in der CI |
| Grafiktreiber schreibt Caches nach `%LOCALAPPDATA%` | Briefing §3.1 nicht voll erreichbar | im Spike messen, ggf. als Grenze dokumentieren |
| Argon2id in reinem Dart zu langsam | Entsperren dauert mehrere Sekunden | eigenes Isolate, Parameter messen, native Alternative prüfen |
| Geschütztes Kopieren unter Linux wird nicht von allen Zwischenablage-Managern beachtet | Link kann im Verlauf landen | dokumentieren, automatisches Leeren als Ergänzung |

---

## 8. Noch offen (wird in späteren Phasen geklärt)

- Konkrete Pakete für SQLCipher, Argon2id, Fensterposition → Phase 1, jedes Paket einzeln zur Freigabe (Briefing §3.1.4)
- Anzeige der Bewertung (0,1er / 0,5er / Sterne) → nach ersten Erfahrungen mit der App
